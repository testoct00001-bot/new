# Deep Hunt Loop — Artifacts

## §1 Small-things sweep checklist

Run this against the target and diff everything against the 404/deny baselines. A delta is a lead until proven otherwise.

**Response shape**
- Status code differs from baseline on a specific path/method (try GET/POST/PUT/PATCH/DELETE/HEAD/OPTIONS on interesting paths).
- Body length differs from baseline or from the SPA shell hash (soft-404 detection).
- Response time noticeably slower/faster (backend touch vs static serve).

**Headers**
- Missing security headers others have (e.g. `X-Frame-Options` absent on one page).
- New `Set-Cookie` on a specific route (session/debug cookies, e.g. `DellCEMSession`).
- `Server`/`Via`/`X-*` changing between paths (mixed stacks behind one domain → separate services).
- `ETag`/`Last-Modified` present → change canary; recheck on a schedule.
- `Content-Type` surprises (JSON on a `.js` path, HTML on an API path).

**Body content**
- HTML/JS comments: TODO, FIXME, DEBUG, internal hostnames, disabled features, old paths.
- Version strings: library versions, build hashes, `__NEXT_DATA__` buildId, chunk hashes.
- Error wording: framework error pages (Spring whitelabel, Tomcat 400, Express `Cannot GET`), verbose envelopes on auth failures — note WHICH host served them.
- Reflected input (params, headers, path segments) → note encoding behavior.
- New/changed files: compare JS bundle hashes run-over-run; new chunks fetched on demand.

**Discovery files and metadata**
- `/robots.txt`, `/sitemap.xml` (+ entries), `/.well-known/*` (openid-configuration, security.txt), `manifest.json`, `favicon.ico` (hash it — shared vs unique).
- TLS cert: issuer, SAN sprawl, expiry (crt.sh for siblings). DNS: CNAME chain, which infra it terminates at (Akamai edge vs direct origin — a structural fact).
- Wayback CDX: archived or never-archived is itself signal.

**Behavioral edges**
- Deny-list exact mapping: which prefixes/patterns are blocked, case behavior, encoding behavior (double-encode, overlong UTF-8, `%uXXXX`, fullwidth/ideographic dots).
- Trailing-dot, trailing-slash, `;` parameters, double slashes, `..` variants — diff edge vs origin handling.
- `Host` mismatch, `X-Forwarded-Host` presence, absolute-URI, `OPTIONS *`, `PRI` — note fail-closed vs fail-open.
- Alias twins: `www.` vs bare vs alternate hostnames — byte-compare; zero divergence is a finding about the infra.
- Auth responses: 401 envelope wording, WWW-Authenticate schemes, unauthenticated listings (providers, tenants, config) — note exactly what is exposed.

## §2 File schemas

### TECHSTACK.MD
| Layer | Hypothesis | Confidence | Evidence |
Each layer (edge, app, JS, origin infra) gets a row; correct rows in place when new evidence lands, never silently.

### KNOWLEDGE.MD
Append-only. Two entry kinds:
- **PATTERN:** name, the find that birthed it, predicted siblings, hit rate.
- **WRITEUP:** trick (one line), source URL, preconditions, target mapping, verdict (UNTRIED/INAPPLICABLE/TRIED-BLOCKED/TRIED-NEGATIVE/VERIFIED), evidence pointer.
End each battery with: counts per verdict + new patterns discovered.

### LEAD.MD
| ID | Lead | Status (OPEN/PARKED/CLOSED) | Evidence | Next step |
PARKED = session-gated (needs credentials, internal vantage, or a deploy change) — kept, never dropped. A lead CLOSES only with evidence.

### FALSEPOSITIVE.MD
| Technique | Why tested | Evidence of negative | Date |
Honest negatives are first-class; they stop the next battery from re-running the same idea.

## §3 Loop state

```
ENUMERATE → OBSERVE(small-things) → PATTERN(predict) → RESEARCH(writeups) → TEST(one trick) → VALIDATE(evidence) → LOG(tables) → back to OBSERVE
```

- Advance on evidence, never on a counter. A battery that produces zero new observations closes its phase; the loop sleeps on canaries (§1 header/TLS/JS-hash) and wakes when one changes.
- Every phase transition asks: "what is the single smallest untested delta?" — that is the next lead.
- When a new JS chunk, new header, ETag refresh, cert change, or deploy appears: re-enter at OBSERVE for the affected area.
