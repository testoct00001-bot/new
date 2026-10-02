---
name: "hidden_endpoint_discovery"
description: "Deep-dig endpoint discovery: the iterative understand → wordlist → probe → understand loop. Find one valid prefix (e.g. /api/), then dig deeper and deeper with target-specific wordlists and small Python probers until hidden files and endpoints surface. Includes response-intelligence if/else tables, a never-stop decision tree, and a full worked example."
metadata: { "includeInPrompt": true }
---

# Hidden Endpoint Discovery — the Deep-Dig Methodology

## Purpose
One valid endpoint is never the end — it is the *door*. This skill is the
discipline of **going deeper from every valid find**: understand what a response
tells you, build a wordlist from *that understanding* (not a generic list),
probe with small Python code, read the new responses, and repeat — until you
are many levels deep in places no scanner ever reaches
(e.g. `/api/` → `/api/v2/` → `/api/v2/data/` → `…/aluserdata.json`).
You stop only when the decision tree says stop, never from boredom.

## The core loop

```
FIND a valid node  →  UNDERSTAND it (response, stack, naming)
  →  BUILD a wordlist from that understanding
  →  PROBE with small Python code (baseline-compared)
  →  UNDERSTAND the new responses
  →  BUILD the next wordlist  →  PROBE again …
```

Every iteration must produce *understanding*, not just hits. A 403 teaches you
as much as a 200 if you read it right (§3).

---

## Phase 1 — First blood: finding the first valid prefix

Seed from evidence, in order:
1. **JS harvesting** — `jsintel.py target.com --subs --crawl`; read `api_client.js`/
   `auth.js` manually for `baseURL` and route tables.
2. **HTML** — comments, forms, `data-*` attributes, `<link>`/`<a>` hrefs.
3. **Discovery files** — `/robots.txt`, `/sitemap.xml`, `/.well-known/*`,
   `/openapi.json`, `/swagger.json`, `/api-docs`.
4. **History** — Wayback CDX for old `*.js`; old bundles name dead-but-live routes.
5. **Guess the obvious once** — `/api`, `/api/v1`, `/graphql`, `/admin`, `/_next`,
   `/static`. One round, baseline-compared, then move to evidence.

Your first valid node is usually a **prefix** (`/api/` → 200/401/403/405), not a
file. That prefix is now your whole world (§2).

---

## Phase 2 — The dig: going deeper from a valid node

This is the heart of the skill. Suppose `/api/` responds non-baseline
(say 403). Do NOT spray 10k generic paths at the root. Instead:

**Step 1 — Learn the node's naming language.** Probe the 6 cheapest children:
`v1 v2 v3 beta internal docs`. Suppose `/api/v2/` → 403 but `/api/v1/` → 404-baseline.
You just learned: **v2 is the live generation.** All future wordlists are v2-first.

**Step 2 — Learn the resource language.** Probe common resource nouns under the
live prefix: `users user account auth login status health config`.
Suppose `/api/v2/users` → 401 and `/api/v2/status` → 200 `{"ok":true}`.
You learned: resource nouns are plural, auth-gated, and `status`-style utility
endpoints exist → add `health metrics info version` to the next wordlist.

**Step 3 — Go one level deeper per valid node.** `/api/v2/users` (401) is valid —
dig *under* it: `me list search export profile settings`.
Suppose `/api/v2/users/export` → 403 with a *different* body length than the
`/api/` 403. Different deny page = different enforcement = interesting.

**Step 4 — Read every body for the next vocabulary.** A 200 on
`/api/v2/config` returning `{"dataRoot":"/data","sheet":"users"}` just handed
you two path segments: `/api/v2/data/` and the word `sheet`. Probe them
immediately — this is how you reach `/api/v2/data/user/sheet/aluserdata.json`:
not by guessing it, but by *following the vocabulary the app gave you*.

**Step 5 — File extensions.** For every valid *directory-like* node, try it as a
file: `/api/v2/data` → `/api/v2/data.json`, `.xml`, `.yaml`. And every valid
*file-like* node gets extension siblings.

**Step 6 — When a level is exhausted, go sideways then down.** Sideways:
synonyms and twins (`user`→`users`→`account`→`member`; `data`→`store`→`records`).
Down: the deepest valid node gets the next round of children.

---

## Phase 3 — Response intelligence: what every response tells you

Record a **baseline** first: 3 random paths (`/zzqx1`, `/zzqx2/nested`) — status,
length, body hash. Every candidate is judged against it.

| Response | Meaning | Do next |
|---|---|---|
| 200, body has JSON keys/URLs | Valid + talkative | Extract every key/URL as next segments (§4) |
| 200, HTML login page | Valid, auth-gated | Note; try `?next=`, method swaps |
| 301/302 | Valid, moved | **Follow it** — destination often reveals the real path; add it as a seed |
| 401 | Valid, needs auth | Record; test without `Authorization` variants; revisit with creds |
| 403 | Valid, forbidden | Run `bypass-403-401` battery; compare deny-page length vs other 403s (different = different layer) |
| 404 but length/hash ≠ baseline | Custom handler = routed | Investigate: try methods, trailing slash, extensions |
| 404 == baseline | Dead… usually | Before abandoning: trailing slash, case variant, one encoding, `.json` |
| 405 | **Valid!** (method rejected = path routed) | Try GET/POST/PUT/PATCH/DELETE/OPTIONS/HEAD |
| 400 | **Valid!** (server parsed it) | Add/fuzz parameters |
| 500 | **Valid and fragile** | Vary input gently; read the error for table/field names |
| 429 | Slow down, don't stop | Reduce threads, continue — rate limit proves the endpoint cares |
| Timeout (others fast) | Possibly doing work | Retry once; note as SSRF/processing candidate |

**The if/else you never skip:** after *every* probe round, classify each result
with the table above and let the classification — not your impatience — choose
the next wordlist.

---

## Phase 4 — Building target-specific wordlists (never generic)

A wordlist is a *hypothesis about this target's naming*. Build from:

1. **Observed vocabulary** — every path segment, JSON key, and filename seen so
   far. Seen `sheet`? Wordlist gets `sheets sheetdata sheetinfo`.
2. **Naming convention** — camelCase? snake_case? kebab? plural nouns? Match it.
   (`userProfile` seen → try `orderHistory`, not `order_history`.)
3. **Resource twins** — `users` ⇒ `orders products invoices tickets members
   roles permissions sessions devices files reports`.
4. **Stack fingerprints** — Spring ⇒ `actuator env beans`; Laravel ⇒ `telescope`;
   Django ⇒ `admin`; Next.js ⇒ `_next/static`; GraphQL ⇒ `graphql/graphiql`;
   WordPress ⇒ `wp-json/wp/v2/users`.
5. **Version/env axes** — `v1 v2 v3 beta staging dev test internal private admin
   api mobile web`.
6. **Doc axes** — `swagger openapi api-docs redoc schema wsdl`.
7. **File axes** — `.json .xml .yaml .yml .txt .bak .old .log .env .config`.
8. **Admin axes** — `admin dashboard console panel manage control`.
9. **Mutation of hits** — every valid segment gets: singular/plural, prefix/suffix
   variants (`get_`, `_list`, `-api`), case variant, trailing-slash twin.

Size: 50–300 per round, regenerated each iteration. A 10k generic list is what
you use when you've stopped thinking.

---

## Phase 5 — Python probe kit (small, copy-paste, baseline-compared)

**prober.py** — the workhorse. Baseline + probe + classify:
```python
import requests, hashlib, sys
BASE = "https://target.com"   # set me
s = requests.Session(); s.headers["User-Agent"] = "Mozilla/5.0"

def sig(r):
    return (r.status_code, len(r.content), hashlib.md5(r.content).hexdigest()[:8])

bl = [sig(s.get(f"{BASE}/zzqx{i}", timeout=10)) for i in (1, 2)]
print("baseline:", bl[0])

def probe(p):
    try:
        r = s.get(BASE + p, timeout=10, allow_redirects=False)
    except Exception as e:
        return print(f"{p} -> ERR {e}")
    s_, l_, h_ = sig(r)
    base_hit = any(s_ == b[0] and h_ == b[2] for b in bl)
    flag = "BASELINE" if base_hit else "*** INTERESTING ***"
    print(f"{s_} {l_:>6} {h_} {p}  {flag}")
    return (p, s_, l_, h_, r)

words = [l.strip() for l in open(sys.argv[1]) if l.strip()]
for w in words:
    probe("/api/v2/" + w)   # point me at the current dig node
```

**classify habit** — after each run, sort output: non-baseline first. Every
`*** INTERESTING ***` line goes through the §3 table *before* the next round.

**dig.py** — recursion driver (pseudo, adapt per target):
```python
# seeds = valid nodes; for each, probe children; recurse into new valid nodes (max depth 6)
seen, depth = set(), {}
def dig(node, d):
    if d > 6 or node in seen: return
    seen.add(node)
    for child in wordlist_for(node):      # §4-built, node-specific
        p, sc, ln, h, r = probe(node + "/" + child)
        if is_valid(sc, ln, h):           # §3 classification
            note_intel(node, child, r)    # harvest vocabulary for next wordlist
            dig(node + "/" + child, d + 1)
```

**Rules for the kit:** threads ≤ 10; `allow_redirects=False` (read 30x manually);
always re-baseline if the app is dynamic; save every round's output — the
*dead* lists stop you re-probing.

---

## Phase 6 — Sensitive file & doc hunting

> Full playbook with per-pattern confirm rules: **`SENSITIVE.md`** (same folder).
> Use it for every sensitive round — it is what keeps this phase false-positive-free.

Once a prefix is valid, hunt the files developers leave beside it:
`swagger.json openapi.json api-docs redoc.html schema.json wsdl`
`/.git/HEAD /.env .env.bak config.json settings.json`
`actuator/health actuator/env` (Spring), `telescope` (Laravel),
`server-status`, `phpinfo.php`, `debug/`, `.well-known/security.txt`.
Each hit is both a finding *and* vocabulary for §4. A `swagger.json` hit ends
the guessing game for that prefix — parse it, probe every documented path.

---

## Phase 7 — Version & environment enumeration

For every valid prefix, enumerate the axes *before* going deep — one valid
`v2` changes where all deep effort goes:
- versions: `v1 v2 v3 v4 beta alpha`
- envs: `dev staging test qa uat sandbox demo internal`
- combos: `/api/v2-internal/`, `/staging-api/v1/` (dashes and subdomains too:
  `api-staging.target.com`, `v2-api.target.com`)
Test subdomains with the same prober — a whole hidden API often lives on
`api-dev.target.com` with zero auth.

---

## Phase 8 — When stuck: the research loop (never stare at a wall)

If two consecutive rounds yield nothing new:
1. **Name what you're stuck on precisely** — "Spring Boot 3, `/api/v2/` valid,
   403 on resources, need bypass/enumeration tricks *for this stack*."
2. **Search it:** Google/GitHub: `spring boot hidden actuator endpoints 2024`,
   `site:github.com <tech> swagger hidden`, HackerOne Hacktivity for the stack.
   Feed every trick into the next wordlist/prober.
3. **Writeup-mine:** `writeup-intel` skill — find a report on the same stack,
   extract the *trick*, adapt it.
4. **Change the axis, not the effort:** stuck going deep? Go sideways (twins),
   up (parent prefixes), or out (subdomains, other ports, mobile API).
5. Only after research + axis change both fail do you park the node — as
   INCONCLUSIVE with notes, never as "done".

---

## Phase 9 — The never-stop decision tree

```
new valid node?
├─ YES → harvest vocabulary → build node wordlist (§4) → probe (§5)
│         └─ deeper valid? → YES: recurse (max depth 6) / NO: go sideways (twins)
├─ NO, but 401/403/405 → bypass battery (bypass-403-401) → still gated?
│         ├─ bypassed → treat as valid, dig
│         └─ still gated → record, revisit with creds; move sideways
├─ NO, all baseline-404 → tried slash/case/encoding/.json variants?
│         ├─ NO → try them (one round)
│         └─ YES → research loop (§8) → new axis → else park as INCONCLUSIVE
└─ 429/timeouts → slow down, continue (never abort on rate limits)
```

---

## Phase 10 — Worked example: `/api/` → `/api/v2/data/user/sheet/aluserdata.json`

The full dig, step by step, with the thinking shown. Mindset first: **every
response is the app telling you where to look next.** You are not guessing —
you are *listening*, then acting on what you heard. Valid thinking only: each
probe must be justified by something you already observed.

**Step 0 — Baseline.** `GET /zzqx1` → 404, 1,204 bytes, hash `a1b2`.
`GET /zzqx2/deep` → 404, 1,204 bytes, hash `a1b2`. Baseline locked: 404/1204/a1b2.
*Thinking: without this, every later judgment is a guess.*

**Step 1 — First blood.** `GET /api/` → **403**, 312 bytes.
Not baseline → valid node. It's a *prefix*, not a file — a door, not a room.
*Thinking: 403 means "I exist, you can't come in." Existence is what I need.
Depth-first starts here, not at the root.*

**Step 2 — Version axis (cheapest question first).** Wordlist: `v1 v2 v3 beta`.
- `/api/v1/` → 404/1204/a1b2 (baseline — dead)
- `/api/v2/` → **403**, 312 bytes (same deny as `/api/` — same layer)
- `/api/v3/` → 404 baseline; `/api/beta/` → 404 baseline.
*Thinking: v2 is the live generation. v1 dead means the old API was removed —
but its* clients *may still exist in old JS bundles (Wayback, §1.4). Every
future wordlist is now v2-first. One round of 4 probes bought me direction.*

**Step 3 — Resource nouns.** Under `/api/v2/`, wordlist from convention
(plural REST nouns): `users user auth login status health config`.
- `/api/v2/users` → **401** (valid, auth-gated — record, revisit with creds)
- `/api/v2/status` → **200** `{"ok":true,"version":"2.4.1"}`
- rest → baseline-404.
*Thinking: three lessons. (a) Nouns are plural. (b) 401s are a whole gated
neighborhood — the auth boundary is* here*, so IDOR tests will live here later.
(c) `status` leaks a version: `2.4.1` → Google "app 2.4.1 changelog/CVE" goes
in the research file. Vocabulary so far: {v2, users, status}.*

**Step 4 — Utility twins.** `status` worked → its siblings exist:
`health metrics info version ping`. `/api/v2/health` → 200 with
`{"db":"ok","cache":"ok"}`. *Thinking: utility endpoints describe the
machine. `db`/`cache` keys go in the vocabulary file — later, `?debug=true`
or `/api/v2/debug` become justified probes, not guesses.*

**Step 5 — Data nouns.** Wordlist from observed language + resource twins:
`data config files reports exports`.
- `/api/v2/data/` → **403**, **187 bytes** — different deny page than the
  312-byte one!
*Thinking: different deny signature = different enforcement layer (maybe WAF
vs app, maybe a different middleware). Two layers means two chances — a bypass
that fails at one may pass the other. Run the bypass-403-401 battery here
later; for now, note it and dig* under *it.*

**Step 6 — Dig under `/api/v2/data/`.** Wordlist: `user users export list`.
- `/api/v2/data/user` → **401**. `/api/v2/data/users` → 404-baseline.
*Thinking: singular `user` is valid, plural is dead — convention broken, good:
the app is inconsistent, and inconsistency is where hidden things live. Note
the anomaly: this team's convention is plural at v2 root but singular under
/data. Future wordlists under /data use singular-first.*

**Step 7 — Dig under `/api/v2/data/user/`.** Wordlist (singular nouns +
data-words): `profile settings sheet sheets export`.
- `/api/v2/data/user/sheet` → **200** `[{"name":"aluserdata","rows":412},
  {"name":"allogdata","rows":98}]`
*Thinking: JACKPOT — but not the end. The response* is *the next wordlist:
`aluserdata`, `allogdata`. Also: why does a "sheet" endpoint exist? Somebody
built a spreadsheet-import feature → hunt its siblings: upload endpoints,
`/import`, `/template`. Vocabulary file grows: {sheet, aluserdata, allogdata}.*

**Step 8 — The file.** `/api/v2/data/user/sheet/` listing is directory-like.
- `/api/v2/data/user/sheet/aluserdata` → 404-baseline (dead as path…)
- *Thinking: dead as a path, but the listing said it exists — so it's a
  FILE. Files have extensions.* → `aluserdata.json` → **200** — the file.
  `{"s3Key":"exports/aluserdata.json","region":"ap-south-1", ...}`.
*If `.json` had failed: `.xml .yaml .csv .txt`, then case variants. Each
failure is one line in the notes, not a reason to stop.*

**Step 9 — Never stop at the file.** The file is a node too:
- Its keys are vocabulary: `s3Key` → is there `/api/v2/data/user/sheet/s3`?
  `region` → multi-region? `/api/v2/data/user/sheet/allogdata.json` (the twin
  from Step 7 — **always take the twin**).
- Its values are leads: an S3 key → bucket existence check (authorized scope!);
  `exports/` prefix → `/api/v2/data/user/sheet/exports/`?
- Sideways: `aluserdata` ⇒ `allogdata` worked ⇒ try `aluserimports`,
  `sheet2`, `backup`. Naming anomalies compound.
- Up one level with new vocabulary: `/api/v2/sheet/`? `/api/v2/exports/`?

**Step 10 — The dead-end protocol (used at every step above).** When a probe
round returns all-baseline:
1. Variants round (one each): trailing slash, case, `.json`, one encoding.
2. Still dead → sideways (twins/synonyms of the *parent's* vocabulary).
3. Still dead → research loop (§8): search the stack + the exact words you
   found (`"aluserdata"`, `"sheet" api`).
4. Still dead → park as INCONCLUSIVE with full notes. The node stays in the
   tree — a new trick or credential reopens it.

**The mindset, compressed:** depth beats breadth (one valid `/api/` fully
excavated beats 10k sprayed paths); responses are instructions (read them
like the app is *trying* to tell you); vocabulary compounds (every hit makes
the next wordlist smarter); twins always exist (developers copy-paste);
dead ends are data (record *why*); and you stop only when the tree (§9) says
INCONCLUSIVE — never because "I tried a few things."

---

## Output Contract
- **Dig tree:** `node → result → what it taught → next wordlist` (keep the chain;
  it *is* the methodology working).
- **Endpoint table:** path → method → status → evidence class
  (OBSERVED / PREDICTED / ROUTED / VERIFIED) → how found.
- **Parked nodes:** INCONCLUSIVE with *why* (what was tried, what research said).
- **Vocabulary file:** every harvested segment/key/filename — future targets
  with the same stack start richer.

## Operating Rules
1. **Baseline everything.** No classification without a control; dynamic apps get
   re-baselined per round.
2. **Evidence-built wordlists only.** Generic 10k lists are the last resort and
   must still be baseline-filtered.
3. **Read every body.** The next wordlist lives inside the last response.
4. **Depth before breadth:** exhaust a valid node (down 6, sideways, variants)
   before opening a new front.
5. **429/timeout = slow down, never stop.** Rate limits prove value.
6. **Two dead rounds → research loop, not surrender** (§8).
7. **Park, don't delete:** INCONCLUSIVE nodes keep their notes; creds or a new
   trick may reopen them.
8. Stay in authorized scope; modest rates (≤10 threads); never destructive.
