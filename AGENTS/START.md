# START — Deep Hunt Loop, Multi-Agent Mode

**Read this file first.** It is the main file. It tells the AI how to use everything in this skill folder to run a never-ending, lead-driven deep hunt on one target.

## What lives here

| File | What it is |
|---|---|
| `START.md` (this file) | The orchestrator. How to run the agents, in what order, with what rules. |
| `SKILL.md` | The loop methodology: phases, baselines, the reportability bar. |
| `references/loop-artifacts.md` | The small-things checklist + the 4 shared table schemas + the loop state diagram. |
| `agents/agent1.md` | **The Mapper** — maps the surface + tech stack in depth, harvests every lead. |
| `agents/agent2.md` | **The Pattern Smith** — turns finds into patterns, predicts sibling endpoints. |
| `agents/agent3.md` | **The Writeup Hunter** — mines writeups/reports/articles/CVEs, extracts reusable tricks. |
| `agents/agent4.md` | **The Precision Tester** — tests one trick at a time, builds raw evidence. |
| `agents/agent5.md` | **The Gatekeeper** — kills false positives, validates findings, feeds the loop back to agent1. |

## Reading order for the AI

1. This file (`START.md`) — the plan.
2. `SKILL.md` — the method.
3. `references/loop-artifacts.md` — the checklist + shared file schemas.
4. `agents/agentN.md` — read the agent's file FRESH every time you run it. Never rely on memory of what an agent does.

## The chain (the loop)

```
agent1 (Mapper) → agent2 (Pattern Smith) → agent3 (Writeup Hunter)
      → agent4 (Precision Tester) → agent5 (Gatekeeper) → back to agent1 …
```

Each agent reads the shared workspace, does its job, writes its outputs, and hands a short brief to the next agent: *what I found, what I closed, what you should try first.* The agents are linked — each one's brief is the next one's starting point, and agent5's restart brief sends agent1 back in sharper than before.

## How to run them

1. **Create the workspace:** `<target>/` containing `TECHSTACK.MD`, `KNOWLEDGE.MD`, `FALSEPOSITIVE.MD`, `LEAD.MD`, and `raw/`.
2. **Read `agents/agent1.md` fully**, then spawn a subagent with that file's contents as its mission, pointed at the workspace. Give it the target host and nothing else it doesn't need.
3. **When it returns its handoff brief**, read `agents/agent2.md`, spawn agent2 with agent2's file + agent1's brief.
4. Repeat through agent3 → agent4 → agent5.
5. **agent5's verdict decides:** new leads → restart at agent1 with the restart brief. Everything closed → the round ends; the loop sleeps on canaries (below).

Run agents strictly in order. Never skip agent5 — it is the quality gate the whole system depends on.

## The shared workspace contract

All five agents read and write the same four files (schemas in `references/loop-artifacts.md` §2):

- `TECHSTACK.MD` — the stack, with confidence + evidence. Corrected in place, never silently.
- `LEAD.MD` — every lead: OPEN / PARKED / CLOSED, with evidence and next step.
- `KNOWLEDGE.MD` — append-only: verified patterns + writeup intel with verdicts and sources.
- `FALSEPOSITIVE.MD` — closed techniques with evidence. Honest negatives are first-class.
- `raw/` — Burp-style raw request/response pairs for anything interesting.

Before probing anything, every agent checks these tables. **Nothing is researched or tested twice without a new reason.**

## Rules that never bend

1. **One host.** And verify which host answered every single time (edge vs origin, SPA shell vs backend API). Misattributed results are contamination, not findings.
2. **Evidence or it didn't happen.** Raw request/response pairs for everything reportable.
3. **No false positive survives agent5.** Findings must be valid and impactful — reproduced 2×, baseline-compared, never faked. Below the bar is below the bar; say so plainly.
4. **Show raw results understood, not pre-filtered.** Agents 1–4 record everything observed; agent5 decides what counts as a finding.
5. **Polite rates.** Modest request volume, no disruptive load, no real user data, observe don't exploit.
6. **Every round must be sharper than the last.** New angles, new patterns, new writeup classes, new predictions. A round that repeats the last round is a failed round — agent5 must call it out in the restart brief.
7. **Small things compound.** One missing header, one changed byte, one odd cookie — the checklist exists because real finds hide in deltas.

## Stop rule (the "never stop" part)

The loop is **lead-driven, not time-driven**. It does not end because N probes ran or an hour passed.

- A round ends only when every lead is CLOSED and a full battery yields **zero new observations**.
- Then the loop sleeps on **canaries**: JS bundle hashes, ETags/Last-Modified, response headers, TLS cert dates, favicon hash. Recheck on a schedule.
- **Any canary change wakes the loop** — agent1 re-enters at OBSERVE for the changed area.
- Session-gated leads (need credentials or an internal vantage point) are PARKED, never dropped and never forgotten.

## Quick-start checklist

- [ ] Workspace created (`<target>/` + 4 tables + `raw/`).
- [ ] `SKILL.md` and `references/loop-artifacts.md` read.
- [ ] agent1 spawned with `agents/agent1.md` as mission.
- [ ] Each handoff brief passed to the next agent with its file.
- [ ] agent5 ran — verdict: restart (new leads) or sleep on canaries.
- [ ] Findings (if any): CONFIRMED only, with raw pairs. Otherwise: "zero reportable" stated plainly.
