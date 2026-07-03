---
name: dependency-audit
description: Vet a new dependency (or a dependency-heavy diff) before it lands. Proves the package exists with the exact spelling on the official registry BEFORE any install, then checks health and supply-chain risk. Use for "should I add this package", "audit these deps", "is this library safe", "review this package.json/requirements.txt/go.mod/Cargo.toml change", or whenever an LLM-suggested import points at a package you have not personally verified exists.
---

# Dependency Audit

You are deciding whether a dependency earns its place in the tree. The cardinal rule: **you may not run an install command until you have proven the package exists on the official registry with the exact spelling you intend to install.** Model-suggested package names are a supply-chain attack surface: LLMs hallucinate plausible names, and attackers pre-register those exact names as malware (slopsquatting). The install command is where a typo becomes a breach — so verification comes strictly before it.

Second rule: **every new direct dependency needs a one-line justification against writing it yourself.** A left-pad-sized need does not justify a tree-sized liability.

## Phase 1 — Existence & typosquat gate (do not emit an install command before this passes)

For the exact string you plan to install, confirm it resolves on the official registry and is the package you think it is. Run the check for the ecosystem, paste the real output.

| Ecosystem | Existence check | Squat/typo check |
|---|---|---|
| npm | `npm view <pkg> version time.created homepage repository` | Compare to the intended repo; watch for scope confusion (`@org/pkg` vs `pkg`), hyphen/underscore swaps, added `js`/`2` suffixes |
| PyPI | `pip index versions <pkg>` or `curl -s https://pypi.org/pypi/<pkg>/json \| jq '.info.name, .info.home_page, .urls[].upload_time'` | Underscore vs hyphen normalize the same on PyPI, but the *import name* differing wildly from the repo is a flag; check `.info.author` and project URLs point to the real project |
| Cargo | `cargo search <pkg>` then `curl -s https://crates.io/api/v1/crates/<pkg> \| jq '.crate.repository, .crate.downloads'` | Look-alike names with swapped words; verify `repository` is the canonical source |
| Go | `curl -s "https://proxy.golang.org/<module>/@v/list"` (module path lowercased) | The module path *is* the identity; a different host/org path is a different package. Verify the path matches the real repo |
| RubyGems | `gem info <name> -r` or `curl -s https://rubygems.org/api/v1/gems/<name>.json \| jq '.name, .source_code_uri, .downloads'` | Underscore/hyphen swaps; verify `source_code_uri` |
| Maven | `curl -s "https://search.maven.org/solrsearch/select?q=g:<group>+AND+a:<artifact>&rows=1&wt=json" \| jq '.response.numFound'` | groupId is the trust anchor — a familiar artifactId under an unfamiliar groupId is a different, untrusted publisher |

Gate conditions — **all must hold** before install:
- The exact name resolves (non-empty, non-404).
- The `repository`/`homepage` points at the project you actually intend (not a fresh empty repo).
- First-publish date and version history are consistent with a real project (a "popular utility" published 6 days ago with one version is a squat until proven otherwise).

If the name does **not** resolve → **stop**. Do not "npm install and see." A non-existent name suggested by a model is the exact slopsquat trigger. Re-derive what the code actually needs and find the real package by reading docs, not by guessing spellings.

**Common squat patterns to eyeball the name against** (any hit means slow down and confirm identity):

| Pattern | Legit | Squat |
|---|---|---|
| Char swap / typo | `requests` | `reqeusts`, `request` |
| Separator swap | `python-dateutil` | `python_dateutil` published by someone else |
| Added filler word | `crossenv` | `cross-env` clone, `colourama` vs `colorama` |
| Scope/namespace drop | `@types/node` | `types-node` |
| Look-alike homoglyph | `lodash` | `1odash`, `l0dash` |
| Suffix/version bait | `axios` | `axios-js`, `axios2` |

## Phase 2 — "Do we even need it?" gate

For each new direct dep, write one line: *what it does that's not worth writing ourselves.*

- Is the used surface one function? (`is-odd`, `left-pad`, a UUID v4, a debounce) → inline it; don't take the tree.
- Does something already in the lockfile cover it? Grep existing deps first.
- Does the stdlib cover it? (`crypto.randomUUID`, `Array.prototype.flat`, `pathlib`, `net/http`.)

No justification line → the dependency does not land.

## Phase 3 — Health signals

Pull these and judge the package as a maintained artifact, not a snapshot.

- [ ] **Last release date** — abandoned (>18mo, open CVEs) or churning-too-fast?
- [ ] **Maintainer count / bus factor** — single maintainer + your prod critical path = risk to log.
- [ ] **Download trend** — flat-tiny or suspiciously spiking (post-typosquat)?
- [ ] **Open critical issues / unpatched advisories** — `npm audit`, `pip-audit`, `cargo audit`, `govulncheck`.
- [ ] **Install scripts present?** — `npm view <pkg> scripts` for `preinstall`/`postinstall`; these run arbitrary code at install time.
- [ ] **Transitive dep count** — `npm view <pkg> dependencies` / check the lockfile delta. 4 lines of need pulling 60 packages is a no.
- [ ] **Repo ↔ registry match** — is the published artifact built from the repo you're reading? (provenance/attestations if available.)

**Reading an install script before you trust it** — when Phase 4 flags `postinstall`/`preinstall` or a `setup.py` that runs code, actually open it (`npm view <pkg> dist.tarball` then extract, or read the sdist). Red flags: base64/hex blobs decoded and `eval`'d, `curl`/`wget` to an IP or pastebin, writes to `~/.ssh` / `~/.npmrc` / env dumps, `child_process`/`os.system` spawning shells, obfuscated one-liners. A legit build script compiles native code or copies files — it does not phone home or read your credentials.

## Phase 4 — Supply-chain risk table

Classify the package on these axes; each "yes" raises the bar for landing it.

| Risk | How to detect | Why it matters |
|---|---|---|
| Install-time scripts | `npm view <pkg> scripts`; `pip` sdist with `setup.py` running code | Code executes on `install`, in CI and dev machines, before your tests ever run |
| Native builds | `gyp`/`node-gyp`, `build.rs`, C extension in sdist | Compiles arbitrary C at install; breaks reproducibility; needs toolchain in prod images |
| Network at runtime | grep the package for outbound calls, telemetry | Phones home; exfiltration surface; availability coupling |
| License family | `npm view <pkg> license`; check SPDX | Copyleft (GPL/AGPL) in a proprietary codebase is a legal landmine; verify against your policy |

## Phase 5 — Pin it so the vetted artifact is the one that installs

Verifying a name means nothing if a later resolve pulls a different build. After a package clears:

- **Pin the exact version and hash.** `package-lock.json`/`yarn.lock`, `requirements.txt` with `==` + `--hash` (or `pip-compile`/`uv`), `Cargo.lock`, `go.sum`, `Gemfile.lock`. Commit the lockfile in the same diff.
- **Prefer `--frozen-lockfile` / `npm ci` / `pip install --require-hashes`** in CI so an attacker who later republishes the same version number (or a compromised registry) can't swap the bytes.
- **Turn off install scripts by default where you can** (`npm ci --ignore-scripts`, then allowlist the few that genuinely need them) — this neutralizes the most common post-typosquat payload vector.
- **Record why a version, not `latest`.** Floating ranges (`^`, `~`, `*`) re-open the supply chain on every install; a pinned version is the artifact you actually audited.

## Auditing a dependency-heavy diff (many deps at once)

When the change is a big lockfile delta rather than one new import, don't hand-check hundreds of lines — bound the work:

```bash
# What actually changed, direct vs transitive
git diff <base>..HEAD -- package-lock.json yarn.lock pnpm-lock.yaml requirements*.txt go.sum Cargo.lock
npm ls --all --depth=0            # or: pipdeptree ; cargo tree ; go mod graph
```

- **Separate direct from transitive.** New *direct* deps get the full Phase 1–5 treatment. New *transitive* deps you scan for the loud risks: brand-new packages, install scripts, name look-alikes to known packages (classic squat vector for indirect deps).
- **Diff the count, not just the names.** A one-line manifest change that adds 200 transitive packages is the finding — flag the blast-radius jump.
- **Run the ecosystem auditor on the delta**: `npm audit`, `pip-audit`, `cargo audit`, `govulncheck ./...`, `osv-scanner`. Treat output as candidates to triage, not verdicts.
- **Any brand-new-and-popular transitive dep is the slopsquat signature** — a package with a familiar name, published days ago, suddenly in your tree. Verify its identity even though nobody "chose" it.

**Beyond typosquats — a package that *exists and is real* can still turn hostile.** Existence checks don't catch: maintainer-account takeover pushing a malicious patch release, a formerly-good package sold to a new owner, or protestware triggered by locale/date. This is why Phase 5 pinning + hashes matter even for trusted names, and why a *new patch version* of an existing dep still deserves a glance at its diff/changelog before you bump it — not just new packages.

## Worked example — agent-suggested package caught at the gate

An agent writes `import requests_async_retry` and suggests `pip install requests-async-retry` for "async HTTP with retries."

Phase 1:

```
$ pip index versions requests-async-retry
WARNING: No matching distribution found for requests-async-retry
$ curl -s https://pypi.org/pypi/requests-async-retry/json | jq .message
"Not Found"
```

Name does not resolve → **gate fails, stop.** This is the slopsquat danger zone: had a typosquatter registered that exact name last week with a `postinstall` payload, "just installing to check" would have run it. Do not try neighboring spellings blindly.

Re-derive the actual need: async HTTP client + retry/backoff. Read real docs → the maintained options are `httpx` (async client, exists since 2019, high downloads, clean repo) plus `tenacity` or `httpx`'s transport-level retries.

```
$ curl -s https://pypi.org/pypi/httpx/json | jq '.info.name, .info.home_page, (.releases|keys|length)'
"httpx"
"https://github.com/encode/httpx"
100+
```

Verdict: drop the hallucinated name; adopt `httpx`, justified because a correct async connection-pooling + retry HTTP client is not worth hand-rolling.

## Output contract

Deliver per package: existence-check output (pasted), the need-justification line (or "inline instead"), the health checklist with values, the risk-table classification, and a verdict `ADOPT / INLINE / REJECT` with the reason. Only after all packages are `ADOPT` do you present the install command. Use this verdict block:

```
pkg: <exact name>@<exact version>
exists: <yes — registry output pasted above>
need: <one line: why not inline>
health: last release <date> | maintainers <n> | transitive <n> | install-scripts <yes/no>
risk: <native/network/copyleft flags, or "none">
verdict: ADOPT | INLINE | REJECT — <reason>
```

Do not emit the install command until every listed package reads `ADOPT`.

## Anti-patterns (hard no)

- Running `npm install` / `pip install` before pasting the existence-check output for that exact name.
- Trying alternate spellings of a name that didn't resolve instead of re-deriving the real package from docs.
- Adding a dependency for a one-function need that inlines in under 30 lines.
- Judging only the direct package and ignoring the transitive weight it drags in.
- Treating a green `npm audit` as sufficient — it covers known CVEs, not abandonment, install scripts, or squats.
- "It's popular so it's fine" without checking the repo↔registry match and publish history.
