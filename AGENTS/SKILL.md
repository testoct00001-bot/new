---
name: "deep_hunt_loop"
description: "Run a never-ending, lead-driven deep hunt on one host: understand the tech stack in depth, harvest every endpoint/directory/file lead plus every small observation, derive new patterns from each find, research writeups/reports/articles for each pattern, test them, and feed results back into the loop. Use when the user wants 'more and more' depth on a single target and refuses to stop at the first pass."
metadata: { "includeInPrompt": true }
---

# Deep Hunt Loop

## Purpose
One target, one loop, no finish line until the surface is genuinely exhausted. This skill orchestrates the other recon skills in a lead-driven cycle: every find becomes a lead, every lead becomes a pattern, every pattern becomes a writeup search, every writeup becomes a test, every test result becomes new leads — repeat.

## Workflow

### Phase 0 — Scope lock and workspace
1. Lock to ONE host. Note the user's folder-per-subdomain convention: create `<target>/` with `TECHSTACK.MD`, `KNOWLEDGE.MD`, `FALSEPOSITIVE.MD`, `LEAD.MD`, and `raw/` for evidence.
2. **Multi-agent mode:** for a full hunt, read `START.md` first — it orchestrates `agents/agent1.md` … `agents/agent5.md` (Mapper → Pattern Smith → Writeup Hunter → Precision Tester → Gatekeeper) in the lead-driven loop. The phases below are what each agent executes.
2. **Host-attribution discipline:** before trusting any response, verify WHICH host answered (SNI vs Host header, edge vs origin, SPA shell vs backend API). Log misattributed results as contamination, never as findings.

### Phase 1 — Deep stack understanding
3. Run `stack-fingerprint` and write the result into `TECHSTACK.MD` with confidence levels and exact evidence. Correct it whenever new evidence contradicts it.
4. Understand the stack, don't just label it: for each layer ask what it implies — its route conventions, its config/debug files, its admin consoles, its known-bad versions, its default paths.

### Phase 2 — Baseline discipline
5. Record the 404 baseline: definitely-nonexistent path, same method, body hash + length. Hash the SPA shell too (soft-404s that serve `index.html` for every route).
6. Fingerprint every deny page (WAF/edge/app-level): exact body, headers, length. A candidate "found" only if it differs from ALL baselines.

### Phase 3 — Surface mining and lead harvest
7. Run `hidden-endpoint-discovery` (JS bundles, source maps, chunks, discovery files, Wayback CDX, npm archaeology). Every real hit goes to `LEAD.MD` as **OPEN**.
8. Run the **small-things sweep** (`references/loop-artifacts.md` §1): status/length/header diffs, comments, version strings, ETags, cookies, error wording, reflections, robots/sitemap, favicon, TLS, DNS CNAME chains, alias byte-compares, deny-list exact mapping. Small deltas are leads, not noise.

### Phase 4 — Pattern derivation
9. From every confirmed find, derive patterns: directory conventions, naming schemes, sibling paths, versioned APIs, framework idioms, parameter names, CRUD tables. Predict and test siblings (e.g. `/api/v1/x` found → try `/api/v2/x`, `/admin/x`, `/x/list`, `/x/export`).
10. Log each pattern in `KNOWLEDGE.MD` as a verified pattern with the find that birthed it.

### Phase 5 — Writeup research loop
11. For each pattern and each stack component, run writeup searches (`browser.search`): `"<component>" bug bounty writeup`, `"<component>" <vuln class> poc`, `site:hackerone.com/hacktivity "<component>"`, CVE lookups for exact versions.
12. Use `writeup-intel`: extract the TRICK (the reusable primitive, not the story), map it to this target's actual stack, test one trick at a time with the matching skill (cors-misconfig, bypass-403-401, otel-abuse, pii-hunter, ssr-renderer-attacks, workers-* …).
13. Log every idea tried in `KNOWLEDGE.MD` with verdict (**VERIFIED** / honest negative) and the source URL — no idea is ever tested twice without a new reason.

### Phase 6 — Validate, log, feed back
14. Run `finding-validator` on anything that looks real: baseline comparison, 2× independent reproduction, Burp-style raw request/response pairs saved to `raw/`.
15. Triage every battery into three piles: **CONFIRMED** (meets the reportability bar) → findings file; **new LEAD** → back to Phase 3/4 with fresh eyes; **negative** → `FALSEPOSITIVE.MD` with evidence so it stays closed.
16. **Feed back:** any new observation (new JS chunk, changed ETag/Last-Modified, new header, new cert, new subdomain pointing here) re-opens the loop at the relevant phase. The loop never ends on a timer — it ends only when every lead is CLOSED and a full battery produces zero new observations.

### Phase 7 — Change canaries
17. Record canaries that signal a re-hunt: JS bundle hashes, ETags, TLS cert expiry/SANs, deploy headers, favicon hash. Recheck them on a schedule; any change restarts the loop.

## Output Contract
- Per battery: counts (probed / new leads / closed / confirmed), the tables updated, `raw/` evidence pointers.
- `LEAD.MD`: every open/parked/closed lead with status, evidence, and next step. Session-gated leads (need credentials or internal vantage) are **PARKED**, never dropped.
- `KNOWLEDGE.MD`: append-only verified patterns + writeup intel with verdicts and sources.
- `FALSEPOSITIVE.MD`: closed techniques with evidence — honest negatives are first-class results.
- A phase ends with the verdict, which may honestly be **zero reportable findings**.

## Operating Rules
1. **Lead-driven, not time-driven.** You stop a phase only when the lead list is exhausted — never because you ran N probes or feel done. Each battery must be sharper than the last: new angles, new patterns, new writeup classes.
2. **Never re-test without a new reason.** Check `KNOWLEDGE.MD`/`FALSEPOSITIVE.MD` before probing anything; the log exists to kill repeat work.
3. **Raw evidence always.** Show raw results understood, not pre-filtered. Save Burp-style request/response pairs for anything reportable.
4. **Reportability bar:** a finding is real only when valid and impactful — reproduced 2×, baseline-compared, no fake predictions. "Zero reportable" is a legitimate, honorable verdict.
5. **Scope discipline:** this skill hunts ONE host. Anything needing another host is a new scope, not a lead.
6. **Polite rates.** Modest request rates, no disruptive load, no real user data, no redeemed URLs/credentials — observe, don't exploit.
7. **Small things compound.** The checklist exists because real finds hide in deltas: one missing header, one changed byte, one odd cookie.
