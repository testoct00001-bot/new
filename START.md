Phase 1 — Map like an architect (30 min, decides everything)

Fingerprint the framework first, not the app. Next.js? Look for __nextDataReq cache behavior. Remix? Probe .data routes. Check /api/v1 vs /v2 — diff them like you did on the Cloudflare SDK; version diffs are where your presign bug lived.
Mine JS bundles for AI surface: /v1/chat, /agent, /mcp, /sse, tools/list. If the app has any AI feature, that becomes your primary target (see Phase 6).
Note every place two systems meet: frontend→backend, app→API gateway, web→mobile API. Bugs live at boundaries now, not in the middle.
Phase 2 — Break the parsers (the new auth bypass factory)

On every login/token/state-changing endpoint: send duplicate JSON keys with conflicting values ({"role":"user","role":"admin"}), double Authorization headers, and Content-Type confusion (JSON body with application/x-www-form-urlencoded and vice versa). Watch which value each layer acts on — frontend says user, backend says admin.
Unicode normalization: usernames, emails, file paths, URL validators. Security check runs before normalization — that's the whole bug class. Try overlong encoding, confusables, combining characters.
Parameter pollution: ?id=1&id=2, scalar vs array (id=1 vs id[]=1), and on PATCH endpoints send nested objects the docs don't mention (mass assignment never died, it just moved to nested JSON).
Phase 3 — Auth layer with fresh eyes (highest payouts live here)

Token confusion: if both a session cookie and JWT exist, which one wins? Revoke one, keep the other, replay.
Password reset: don't just test token guessing — check if the reset token leaks into analytics/referrer (the HackerOne Segment bug: token sent to Segment on page load), Host-header poisoning on the reset link, and whether the token stays valid after use.
OAuth/OIDC non-happy paths: PKCE downgrade (remove code_challenge), redirect_uri parser differentials between the authorize endpoint and the token endpoint, pre-registered vs dynamic clients.
Cookie prefix bypasses (2025 research): __Host-/__Secure- can be defeated — test cookie tossing from subdomains.
Phase 4 — Race everything that touches value

Single-packet race (Burp Repeater, send group in parallel, 1 group): coupons, referrals, wallet credits, votes, rate limits, "claim" buttons. If two requests can both succeed, that's money.
Workflow surgery: skip steps (go straight to step 3), reorder them, replay a completed step. 2FA setup → disable flows, email-change → skip re-auth, checkout → replay the payment callback.
Phase 5 — Server-side, new primitives only

SSRF via redirect loops: point it at your server returning 301→302→303→…→310 chains ending at 169.254.169.254. Error-handling mismatches flip blind SSRF into full response read.
HTTP/2 CONNECT: test if the origin accepts CONNECT over H2 — instant SSRF + internal port scan.
Error-based SSTI: (1/0).zxy.zxy as a universal polyglot in every template-ish input (emails, PDFs, filenames, notification text). Verbose errors = data leak, division-by-zero = boolean oracle.
ORM leak: probe filter syntax — ?filter[email][$ne]=x, resetToken[not]=E. If the API echoes filterable fields, binary-search secrets.
Cache: unkeyed headers (especially X-Forwarded-*, Origin), and on Next.js targets the __nextDataReq trick.
Phase 6 — If the site has AI, drop everything else

Can the agent act? (send email, read files, call internal APIs). If yes: indirect prompt injection via uploaded document, pasted URL, or support-ticket content → get it to exfiltrate via markdown image or to call a privileged tool.
Exposed MCP/SSE endpoints without auth = critical, full stop. Connect and list tools.
The IQ layer — how top hunters actually think

Attack the assumption, not the code. Ask "what does the developer believe is impossible here?" — then do exactly that.
Follow one piece of data. Pick the most sensitive object (admin token, another user's file, payout record) and trace every path to it. Don't hunt vuln classes; hunt the data.
Diff everything. Two roles × every endpoint, diffed byte-by-byte. v1 vs v2. Mobile vs web. Staging vs prod. The difference is the bug.
Read the framework source, not the app. The app is usually fine; the framework's edge cases aren't.
New feature = new bugs. Watch JS bundle diffs and changelogs. Your best odds are code less than 90 days old.
Never stop at the first bug in a flow. The second bug in the same flow is usually the critical one — the dev fixed the obvious path and left the twin.
Chain, don't report singles. A low alone is noise; low + low with a path to data or money is a high.
