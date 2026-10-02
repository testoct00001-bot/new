# SENSITIVE.md — hunting sensitive files, endpoints & directories

Companion to `SKILL.md`. The deep-dig loop finds *things*; this file tells you
how to aim it at the things that **matter** — credentials, PII, auth material,
configs, backups — and how to confirm each one so you never report a false
positive. A 200 on `.env` that returns the SPA shell is not a finding. It's
embarrassment. Every pattern below carries its **confirm rule**: the exact
content check that separates real from pathetic.

## 1. Sensitive tiers (hunt in this order)

- **Tier 0 — credentials & keys:** `.env`, `config.json` with secrets, `.git/`,
  private keys, cloud credentials, `secrets.yaml`. Direct takeover material.
- **Tier 1 — auth & session:** token endpoints, `jwks.json` misuse, session
  dumps, password reset internals, SSO metadata with secrets.
- **Tier 2 — PII / data:** user exports, `aluserdata.json`-class files, backups
  (`.sql`, `.bak`), logs with tokens, analytics dumps.
- **Tier 3 — config & docs:** `swagger.json`/`openapi.json` (maps the whole
  attack surface), actuator dumps, `phpinfo`, debug toolbars, `.well-known`.
- **Tier 4 — source & history:** source maps with secrets, `.git/` objects,
  backup copies (`index.php.bak`), old bundles with live endpoints.

## 2. Focusing the deep-dig loop on sensitive targets

The SKILL.md loop is directionless until you point it. Point it like this:

1. **Every valid node gets a sensitive-files round first.** Before digging
   deeper for *endpoints*, fire the sensitive-file wordlist at the node
   (`/api/v2/` → `swagger.json`, `.env`, `config.json` …). Sensitive files sit
   *beside* endpoints, and one hit re-maps everything.
2. **Vocabulary → sensitive axes.** Found the word `sheet`? Sensitive axis adds:
   `sheet_backup`, `sheet_old`, `export`, `dump`, `backup`. Found `config`?
   Add `config.json`, `config.bak`, `config.prod.json`, `.env`.
3. **Extension round on every valid file-like node** (§2.5 of SKILL.md):
   `.json .xml .yaml .yml .txt .bak .old .orig .swp .log .sql .env`.
4. **Parent walk:** sensitive files often live *above* the API:
   `/api/v2/.env`, `/api/.env`, `/.env`. Walk up.
5. **Twin rule for sensitive hits:** `swagger.json` found → try `openapi.json`,
   `api-docs`, `redoc.html` immediately. `.env` found → `.env.bak`,
   `.env.production`, `config.json`.

## 3. The catalog — pattern, why it matters, confirm rule

**Format:** `PATH PATTERN` — why sensitive — **CONFIRM:** content must contain…

### Credentials & secrets
- `/.env`, `/api/.env`, `/api/v2/.env` — app secrets —
  **CONFIRM:** body has `KEY=VALUE` lines (`DB_PASSWORD=`, `AWS_SECRET`), NOT HTML.
- `/config.json`, `/settings.json`, `/appsettings.json` — configs often embed secrets —
  **CONFIRM:** parses as JSON AND has secret-ish keys (`password`, `secret`, `key`, `token`).
- `/.git/HEAD` — repo exposure —
  **CONFIRM:** body starts with `ref: refs/heads/`. Then walk: `/.git/logs/HEAD`,
  `/.git/config` (CONFIRM: `[core]` section).
- `/*.pem`, `/*.key`, `/private.key`, `/id_rsa` — private keys —
  **CONFIRM:** `-----BEGIN .*PRIVATE KEY-----`.
- `/secrets.yaml`, `/vault.json` — secret stores —
  **CONFIRM:** YAML/JSON parse + secret keys.

### Auth & session
- `/.well-known/jwks.json`, `/jwks.json` — signing keys (only sensitive if private
  material or `x5c` with weak alg hints) —
  **CONFIRM:** JSON with `keys` array; note `alg`/`use`. Public keys alone = info.
- `/actuator/env`, `/actuator/configprops` (Spring) — env dumps with passwords —
  **CONFIRM:** JSON `propertySources`, values not all `******`.
- `/api/auth/refresh`, `/api/token/refresh` — token endpoints —
  **CONFIRM:** actually mints tokens (test with a low-priv session), not just 200.

### PII & data
- `*export*.json/csv`, `*dump*.sql`, `*backup*.bak`, `/api/v2/data/user/sheet/*.json` —
  **CONFIRM:** body contains record-like data (names, emails, IDs) — synthetic
  test data you created counts as proof of *access*, real user data must NOT be
  exfiltrated; one redacted sample suffices.
- `*.log`, `/logs/`, `/debug.log` — tokens/PII in logs —
  **CONFIRM:** log lines with `token=`, `password=`, emails.
- `/api/users/export`, `/admin/export` — export endpoints —
  **CONFIRM:** returns *another user's* data (two-account test), not just yours.

### Config & docs (surface multipliers)
- `/swagger.json`, `/openapi.json`, `/v2/api-docs`, `/api-docs` —
  **CONFIRM:** parses as JSON AND top-level has `openapi` or `swagger` key.
  Then probe every documented path — this is the map, not the treasure.
- `/swagger-ui/`, `/redoc.html` — doc UIs —
  **CONFIRM:** HTML references a spec URL; fetch the spec.
- `/actuator`, `/actuator/health` — Spring Boot —
  **CONFIRM:** JSON with `status`. `/actuator/env` needs the Tier-0 confirm.
- `/phpinfo.php`, `/info.php` — full config disclosure —
  **CONFIRM:** `phpinfo()` output table.
- `/server-status`, `/server-info` (Apache) — CONFIRM: `Apache Status`.
- `/telescope`, `/horizon` (Laravel) — CONFIRM: the actual UI/API, not a redirect.
- `/.well-known/security.txt`, `/security.txt` — CONFIRM: `Contact:` line (info only).

### Source & history
- `/*.js.map`, `/app.js.map` — source maps —
  **CONFIRM:** JSON with `sourcesContent` or `sources[]`; grep it for
  `password|secret|api_key` — a map with secrets is Tier 0.
- `/*.bak`, `/*.old`, `/*.orig`, `/*~`, `/*.swp` — backup copies of live files —
  **CONFIRM:** body differs from the live file AND contains code/config, not HTML.
- `/Dockerfile`, `/docker-compose.yml` — build secrets —
  **CONFIRM:** `FROM`/`services:` content with env secrets.

## 4. The false-positive kill list (memorize this)

1. **SPA fallback:** 200 with `text/html` containing `<div id="root">` / the
   app shell = soft-404. Hash it in the baseline; any "sensitive file" with
   that hash is dead.
2. **Content-Type lie check:** `.env` returning `text/html` → read the body.
   If it's HTML, it's not an env file.
3. **Empty-success:** 200 with `{}` / `[]` / `null` = endpoint exists, nothing
   sensitive disclosed. Note as ROUTED, not VERIFIED-sensitive.
4. **Example/placeholder data:** `DB_PASSWORD=changeme`, `example.com` emails,
   `sk_test_123` → the file is real but the secrets are fake. Report the
   *exposure class*, mark secrets unverified.
5. **Auth-gated ≠ accessible:** 401/403 on `/actuator/env` is a *lead*, not a
   finding. It becomes a finding only via bypass or credentials.
6. **Your-own-data:** export endpoints returning *your* account's data prove
   function, not IDOR. Two accounts or it didn't happen.
7. **WAF block pages:** a 403 body saying "blocked" on `/.env` doesn't mean the
   file exists — compare against the 403 for a random path. Same = WAF, not a hit.
8. **Case:** `/.ENV` → 200 on a case-insensitive server while `/.env` → 404
   doesn't make it more real. One canonical confirmation is enough.

## 5. Validation protocol (per hit, no exceptions)

1. **Fetch raw** (no redirects followed blindly — a 302 to `/login` is not a hit).
2. **Apply the CONFIRM rule** from §3. Fails → NEGATIVE, note why.
3. **Prove sensitivity with your own data:** plant a canary
   (unique string in your profile/export), then find the canary in the
   sensitive output. Your canary in someone else's export path = IDOR proven,
   zero real-user data touched.
4. **Reproduce 2×** (fresh session where applicable).
5. **Classify evidence:** OBSERVED (returned bytes) → VERIFIED (confirm rule
   passed + canary/2×). Never file below VERIFIED.

## 6. Worked mini-example

`/api/v2/` valid (403, non-baseline). Sensitive-files round:
- `/api/v2/swagger.json` → 200, `application/json`, body has `"openapi":"3.0.1"`
  → **CONFIRM passed.** Parse it: 47 paths. Feed all 47 into the dig loop.
- `/api/v2/.env` → 200, `text/html`, body hash == SPA baseline → **kill list #1:
  false positive**, note and move on.
- `/.git/HEAD` → 200, `ref: refs/heads/main` → **CONFIRM passed, Tier 0.**
  Next: `/.git/logs/HEAD` (commit history), `/.git/config` — each with its
  own confirm rule. Do NOT clone the repo (scope/noise); targeted object reads only.
- `/api/v2/data/user/sheet/aluserdata.json` → 200 with record-like rows.
  Canary test: update your profile name to `canary_zq9`, re-fetch — canary
  present → **VERIFIED sensitive data exposure.** Redact in the report.

## 7. Never-stop tree (sensitive edition)

```
sensitive-file round at node → hit?
├─ YES → confirm rule (§3) → VERIFIED? → harvest its contents as vocabulary
│         → twins + extensions + parent-walk → keep digging
├─ NO → all baseline? → variants round (case, trailing dot, %2e) → still no?
│         → research loop: "<stack> exposed .env 2024", "<tech> swagger hidden"
│         → new axis (subdomain, /internal/, /admin/) → else park INCONCLUSIVE
└─ 401/403 on the sensitive path → bypass battery → still gated?
          → YES: record as gated-lead (revisit with creds), move on
          → NO (bypassed): run confirm rule on the bypassed response
```

## 8. Output contract

Per sensitive target: path → tier → raw status/headers → **confirm rule result
(PASS/FAIL + evidence snippet, redacted)** → evidence class → blast radius
(what an attacker gets: RCE? takeover? PII of N users?) → twin/extension
follow-ups tried. A sensitive finding without a PASS on its confirm rule
does not exist.

## 9. Time-pass filter — what to IGNORE (protect your hours)

Most recon output is noise. Run every finding through three questions:
**Does it LEAK something? Is it SENSITIVE? Is it reachable UNAUTH?**
Zero yeses → time-pass. Note it in one line and move on. Never dig there.

**Classic time-pass (do not chase):**
- Marketing pages, blogs, docs, changelogs, pricing — public by design.
- `robots.txt`/`sitemap.xml` entries pointing at public pages — recon signal only.
- Version strings alone (`X-Powered-By`, `status.version`) — a version is not a
  vuln; it becomes work only when a CVE/research trick maps to it.
- `/.well-known/security.txt`, `favicon.ico`, `manifest.json` — info, not leads.
- Public API docs describing public endpoints — the *docs* aren't sensitive;
  the *undocumented* paths inside them might be (§6 worked example).
- Your-own-data endpoints (profile, my-orders) — function, not finding.
- WAF block pages, generic 403s identical to baseline — the wall, not the door.
- Third-party scripts (analytics, chat widgets) — note for telemetry-leak
  checks (Phase 9 of hunt-methodology), then move on; don't audit their code.
- Open-redirect on a logout link, `X-Frame-Options` missing on a blog page,
  `ACAO: *` on public JSON — severity-zero; batch them, never chase singly.
- Subdomains that are pure marketing mirrors — one request to confirm, then out.

**The triage habit (30 seconds per finding):**
```
LEAK? ──→ does it disclose data/config/code I shouldn't see?
SENSITIVE? ──→ credentials / PII / auth material / business data?
UNAUTH? ──→ reachable without a session (or with a low-priv one)?
2+ yeses → DIG (this file). 1 yes → one verification round, then park.
0 yeses → time-pass. Log it, close it, keep moving.
```

Time-pass also includes *your own behavior*: re-probing dead nodes without a
new trick, spraying generic wordlists "just in case", screenshotting public
pages as "evidence". The skill's decision trees exist to stop exactly this.

## 10. From lead to access — deep exploitation thinking

A lead is a locked door you've *located*. Now think like someone who wants in.
For each lead class, the access ladder — try in order, stop at first VERIFIED:

**Lead: unauth 200 with data-ish body (exports, sheets, configs)**
1. Canary test: plant unique string in your account → re-fetch. Present?
   → access to the data layer proven.
2. IDOR ladder: swap IDs (`/sheet/aluserdata.json` → `allogdata.json` → other
   users' named files). Enumerate via the listing endpoint if one exists.
3. Parameter ladder: `?user_id=`, `?all=true`, `?admin=1` — mass-assignment
   thinking on read paths.
4. If values are pointers (S3 keys, URLs, file paths) → follow each pointer
   (bucket reachable? URL fetches without auth? path traversal `../`?).

**Lead: 401/403 on something sensitive-looking**
1. Bypass battery (`bypass-403-401`): methods, headers, path confusion, encodings.
2. Auth-confusion: no `Authorization` header vs garbage token vs low-priv token —
   does the gate distinguish, or does *any* token shape change the response?
3. Verb/param smuggling: `POST` with `_method=GET`, `?access_token=` variants.
4. Still gated → credentialed revisit list. A gated Tier-0 (e.g. `/actuator/env`
   403) is your #1 target the moment you hold *any* session.

**Lead: swagger/openapi/doc hit**
1. Parse every path; sort by: no-auth-required markers, admin-ish names,
   deprecated versions.
2. Probe each undocumented-by-UI path unauth with the §3 confirm mindset.
3. Schemas leak field names → those names become wordlist vocabulary AND
   mass-assignment candidates.

**Lead: `.git/` exposure**
1. `HEAD` → branch; `config` → remotes (info); `logs/HEAD` → commit messages
   (often name files/routes).
2. Targeted object reads only — no clone. Each object gets a confirm rule
   (is it code/config with secrets, or time-pass?).

**Lead: source map with code**
1. Grep for secrets first (Tier 0), then route tables (new endpoints),
   then auth logic (how tokens are stored/sent — XSS impact math).
2. Old/duplicate maps (`.map.bak`) → diff against current for removed endpoints.

**Lead: backup file (`.bak`, `.old`, `~`)**
1. Confirm rule: differs from live file + contains code/config.
2. Diff it against the live version — the *delta* is where secrets and old
   routes hide.

**Lead: version string / stack fingerprint**
1. Not a finding — a research seed. Search: `<stack> <version> hidden
   endpoints`, `<stack> <version> CVE`, HackerOne hacktivity for the stack.
2. Every trick found → adapted probe within the hour, or the research was
   time-pass too.

**The access mindset, compressed:** a lead is a *question* ("can I read this
without auth?"). Answer it with the cheapest test first, escalate only on
evidence, and every answer — yes or no — feeds the vocabulary file. Leads
never die; they wait (gated-lead list) for the key: a bypass, a credential,
a new trick from research.

## 11. Operating rules

1. **Confirm rule or it didn't happen.** Every pattern in §3 has one; no
   exceptions, no "looks sensitive".
2. **Canary-first on data:** prove access with your own planted data before
   touching anything resembling real user data — and then don't.
3. **Redact in reports:** one redacted sample proves the class; full dumps
   prove nothing extra and create liability.
4. **Gated ≠ found.** 401/403 is a lead with a revisit plan, not a finding.
5. **Scope discipline:** `/.git/` reads are surgical (HEAD, config, logs) —
   no recursive cloning, no mass object enumeration.
6. **Rate discipline:** ≤10 threads; sensitive rounds are small and smart, not big.
