---
name: security-sweep
description: Code-security review pass over a diff or module — enumerate entry points and trust boundaries, then flag only findings with a concrete attack path. Use for "security review", "is this safe", "check this for vulnerabilities", "review this endpoint/handler/auth", or before merging code that touches input parsing, auth, file paths, queries, or outbound requests. Not a pentest; no live exploitation.
---

# Security Sweep

You are reviewing code for security defects, not running a penetration test. The cardinal rule: **you may not flag a finding until you can name the entry point, the trust boundary it crosses, and the concrete attack path an attacker walks.** A category name ("possible XSS") is not a finding. An attacker's steps are.

Most machine-generated "security reviews" fail two ways: they dump linter/SAST categories with no reachability, or they demand "add validation" without saying what validates what against which threat. Both waste the reader's trust budget. This skill forces reachability before flagging.

## Phase 0 — Recon (enumerate the attack surface fast)

Build the entry-point inventory with grep before reading anything deeply — you can't map boundaries you haven't found.

```bash
# HTTP routes / handlers
grep -rEn "@(app|router|Get|Post|Put|Delete)Mapping|app\.(get|post|put|delete)\(|@(app|router)\.(get|post)|router\.(HandleFunc|Get|Post)" .
# Deserialization / template sinks
grep -rEn "pickle|yaml\.load|ObjectInputStream|render_template_string|Marshal\.load" .
# Outbound + filesystem + shell sinks
grep -rEn "requests\.|fetch\(|urlopen|open\(|readFile|exec|subprocess|os\.system" .
```

Scope to the diff when reviewing a change: `git diff --name-only <base>..HEAD` then run the greps against those files. Reviewing a diff means the *changed* surface — but include callers of changed functions, since a new caller can reach an old sink.

## Phase 1 — Map entry points and trust boundaries (gate: do not flag anything before this exists)

Produce this map first. If you cannot fill it, you cannot review — you're guessing.

- **Entry points**: every place attacker-influenced data enters the code under review. HTTP routes/params/headers/body, CLI args, env, message-queue payloads, file uploads, webhook bodies, deserialized blobs, DB rows written by other tenants.
- **Trust boundaries**: where data crosses from lower to higher trust. e.g. `request body → SQL`, `user id in JWT → resource lookup`, `URL param → outbound fetch`, `uploaded filename → filesystem path`.
- **Sinks**: dangerous operations reachable from entry points — query execution, shell exec, file open, HTTP client, template render, deserializer, `eval`.
- **Assets**: what an attacker wants — other users' rows, secrets, RCE, internal network.

Output as a short table `entry point → boundary crossed → reachable sink → asset at risk`. Every finding in later phases MUST trace to a row here. A finding with no row is theoretical — drop it or find the row.

## Phase 2 — Probe the top vulnerability classes

For each row in the map, run the relevant probes. Grep locates candidates; **reachability from a real entry point promotes a candidate to a finding.**

| Class (OWASP 2025) | What it is | Detection probe |
|---|---|---|
| Injection (SQL/NoSQL/cmd) | Attacker data reaches an interpreter as code | `grep -rEn "execute\(|query\(.*\+|f\"SELECT|format\(.*SELECT|\$\{.*\}.*(SELECT|WHERE)|os\.system|subprocess.*shell=True|exec\("` |
| Broken authz / IDOR | Resource fetched by attacker-supplied id without ownership check | Find route handlers; for each, check the id in the path/body is scoped to the caller (`WHERE owner_id = current_user`). Grep `findById\(|get\(.*params|Objects\.get\(.*id=` then read the surrounding auth check |
| SSRF | Outbound request to an attacker-controlled URL/host | `grep -rEn "requests\.(get|post)\(|fetch\(|http\.get|urlopen|axios\.|HttpClient" ` then check the URL derives from user input without allowlist |
| Secrets handling | Hardcoded creds, secrets in logs/errors, tokens in URLs | `grep -rEn "(api[_-]?key|secret|token|password|passwd|BEGIN (RSA|EC) PRIVATE)" ` and `grep -rEn "log.*(token|password|secret)"` |
| Insecure deserialization | Untrusted bytes → object graph | `grep -rEn "pickle\.loads|yaml\.load\(|Marshal\.load|ObjectInputStream|readObject|unserialize\(|JsonConvert.*TypeNameHandling"` |
| Path traversal | User input in a filesystem path | `grep -rEn "open\(.*(request|params|req\.)|readFile\(.*req|Path\(.*input|os\.path\.join\(.*request|sendFile\("` then check for `..` normalization / base-dir containment |

For each candidate: read the code from entry point to sink. If attacker data reaches the sink without an effective control in between, it's a finding. If a control exists, note it and move on — do not flag defended code.

**What counts as an effective control** (so you don't wave through fake ones):
- Parameterized query / prepared statement — yes. String concatenation with an escaping helper — treat as broken until proven; escaping is context-specific and easy to misapply.
- Server-side authorization predicate scoped to the caller — yes. A client-supplied `role` field or a hidden form field — no, attacker controls it.
- Allowlist of exact hosts/paths — yes. A denylist / regex trying to block `..` or `localhost` — usually bypassable (`0x7f.0.0.1`, `[::1]`, DNS rebinding, `....//`).
- Framework autoescaping left on — yes. `mark_safe`/`html_safe`/`dangerouslySetInnerHTML` on attacker data — no.

## Phase 3 — Framework footguns

These defaults bite even when the general code looks fine.

| Framework | Footguns (check specifically) |
|---|---|
| Django | `.raw()` / `.extra()` / `RawSQL` bypass the ORM's parameterization; `mark_safe` / `\|safe` disables autoescaping; `DEBUG=True` leaks settings + stacktraces; `SECRET_KEY` in source |
| Express | No default body-size limit (DoS); missing `helmet`; `res.sendFile`/`res.download` with unsanitized path; regex-based routes → ReDoS; trusting `X-Forwarded-For` for authz |
| Spring | Insecure deserialization via Jackson polymorphic typing (`enableDefaultTyping`/`@JsonTypeInfo`); SpEL injection in `@Value`/`@PreAuthorize`; mass assignment via `@ModelAttribute` binding to entities; actuator endpoints exposed |
| Rails | Mass assignment when strong params are skipped (`permit!`); `send`/`constantize`/`public_send` on user input; `find_by_sql`/string interpolation in `where`; `raw`/`html_safe` in views |
| Flask/FastAPI | `render_template_string` with user data → SSTI (Jinja `{{7*7}}`); `debug=True` gives an RCE console; missing `Depends()` auth on a route; `pickle`-based session/cache |
| Next.js/Node | Server actions / API routes trusting client-supplied ids; `dangerouslySetInnerHTML`; SSRF via image optimizer / server fetch of user URLs; secrets leaking into the client bundle (non-`NEXT_PUBLIC` assumed safe but imported client-side) |
| Go | SQL via `fmt.Sprintf` into `db.Query`; `template.HTML`/`text/template` for HTML (no autoescape) → XSS; `exec.Command` with a shell string; missing context timeout enabling SSRF-driven resource exhaustion |

## Phase 4 — Severity and output (gate: this exact format)

Rate each finding by **exploitability × blast radius**:

- **Exploitability**: unauth remote (high) → authed remote (med) → needs local/specific state (low).
- **Blast radius**: full DB / RCE / all tenants (high) → single account / limited data (med) → info leak only (low).

| | Radius: High | Radius: Med | Radius: Low |
|---|---|---|---|
| **Exploit: High** | CRITICAL | HIGH | MEDIUM |
| **Exploit: Med** | HIGH | MEDIUM | LOW |
| **Exploit: Low** | MEDIUM | LOW | INFO |

Downgrade rules that keep the report honest: an issue reachable only with credentials the attacker would already need to obtain the asset is INFO, not a finding. A "vulnerability" requiring physical access, an already-compromised admin, or DB access is out of scope for a code review — say so and move on.

Emit every finding in exactly this line format:

```
[SEV] finding — attack path — fix
```

- `attack path` = the concrete steps: which request, which value, what it reaches, what the attacker gets.
- `fix` = what control goes where, naming the validator and the thing validated.

## Worked example — IDOR in a route handler

Code under review (`routes/invoices.js`):

```js
router.get('/invoices/:id', requireLogin, async (req, res) => {
  const inv = await Invoice.findByPk(req.params.id);   // <-- no owner scoping
  res.json(inv);
});
```

Phase 1 map row: `GET /invoices/:id (path param id) → user-trust → Invoice.findByPk → other tenants' invoice data`.

Reachability: `requireLogin` proves a session exists but never checks the invoice belongs to that session's user. Any logged-in user can enumerate ids.

```
[HIGH] Broken authz / IDOR in GET /invoices/:id — log in as user A, request GET /invoices/1002 (an id owned by user B); handler calls Invoice.findByPk(1002) with no ownership predicate and returns B's invoice; attacker scrapes all invoices by incrementing id — fix: scope the query to the caller: `Invoice.findOne({ where: { id: req.params.id, ownerId: req.user.id } })` and 404 on miss so ids aren't oracle-able.
```

Note the shape: named entry point, the exact request, the missing predicate, the escalation to bulk, and a fix that says *which predicate scopes which query*.

## Worked example — SSRF in an "import from URL" feature

Code under review (`services/import.py`):

```python
@app.post("/import")
def import_from_url(req):
    url = req.json["source_url"]          # attacker-controlled
    data = requests.get(url, timeout=5)   # <-- server fetches it
    return parse(data.content)
```

Phase 1 map row: `POST /import (source_url) → user-trust → requests.get → internal network / cloud metadata`.

Reachability: `url` flows straight into `requests.get` with no host allowlist. The server, not the attacker, makes the request — so it reaches things the attacker can't reach directly.

```
[HIGH] SSRF in POST /import — attacker sends {"source_url":"http://169.254.169.254/latest/meta-data/iam/security-credentials/"}; the server fetches the cloud metadata endpoint from inside the VPC and returns the IAM role credentials in the parsed response; attacker then uses those creds against the cloud API — fix: resolve the host and reject non-public IPs (block link-local 169.254.0.0/16, RFC1918, loopback, ::1) via an allowlist of permitted hosts checked AFTER DNS resolution, and disable redirects so a public host can't 302 to an internal one.
```

Note: the fix blocks *after resolution* — checking the string before DNS lets `attacker.com → 169.254.169.254` (rebinding) slip through.

## Output contract

Deliver: (1) the Phase 1 entry-point/boundary table; (2) findings as `[SEV] finding — attack path — fix` lines, ordered by severity, deduplicated (one root cause hit from three routes is one finding with three paths, not three findings); (3) an explicit "checked and defended" list so the reader knows what you looked at and cleared; (4) anything you couldn't reach a verdict on and why (missing code, unclear caller). Lead with CRITICAL/HIGH; never bury an exploitable finding under a pile of INFO.

## Anti-patterns (hard no)

- Flagging a class ("SQL injection possible") without tracing attacker data from a named entry point to the sink.
- "Add input validation" / "sanitize this" without naming what validates what and against which attack.
- Pasting linter or SAST output as findings — those are candidates; unreachable ones are noise.
- Listing defended code as a finding because the pattern matched a grep.
- Inventing an attack that requires the attacker to already have the asset (secret, admin, DB access) — that's not an escalation.
- Severity by vibes — every SEV must come from the exploitability × blast-radius rubric.
