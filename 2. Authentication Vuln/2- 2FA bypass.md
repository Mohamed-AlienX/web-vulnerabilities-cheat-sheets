

# 🔐 MFA / 2FA Bypass — Master Cheat Sheet

**Authentication Logic Testing for Bug Bounty & AppSec**

---

## 🧠 Mental Model — Ask These Before You Start

|Question|Why it matters|
|---|---|
|Is the OTP/token **bound to the user account**?|If not → cross-account bypass is possible|
|Is the OTP **single-use** or reusable?|If reusable → replay attacks work|
|Is OTP verification **rate-limited**?|If not → brute-force is trivial|
|Is 2FA enforced on **every** auth path (password, OAuth, password-reset, magic link)?|Inconsistent enforcement = bypass via the weakest path|
|Are **all active sessions invalidated** when 2FA is enabled or the password is changed?|If not → a previously hijacked session stays valid forever|
|Does the **client** or the **server** make the final access decision?|If the client decides → trivial response/status manipulation|

---

## How This Sheet Is Organized

Each finding has: **Risk Rating** → **What it is** → **Step-by-step PoC** → **Burp/tool tip** → **Impact note**.

Severity is based on typical real-world impact (account takeover potential, ease of exploitation, blast radius). Actual severity always depends on the specific program's risk model — adjust your report accordingly.

---

# 🔴 CRITICAL

These typically lead to direct, low-effort account takeover.

## 1. Cross-Account OTP / Token Usage

**What it is:** The server validates that a code _exists and is unused_, but never checks that it belongs to the _same user_ who is trying to log in. The OTP/token isn't bound to the session or user ID.

**Steps:**

1. Open two test accounts: Account A (attacker-controlled) and Account B (victim/target).
2. On Account B, trigger the login flow up to the OTP step. Do **not** submit the OTP yet.
3. On Account A, request a fresh OTP and capture it (you legitimately own this code).
4. Go back to Account B's pending verification request in Burp Repeater. Replace B's OTP value with A's captured code.
5. Forward the request.
6. ✅ If Account B's session is verified using A's code → **Critical: OTP not bound to user**.

**Tip:** Also test this with the **partial auth token** itself (not just the numeric OTP) — some apps issue a token after step 1 that should map 1:1 to a user but doesn't.

**Impact:** Any authenticated low-privilege user can generate valid codes for themselves and use them to satisfy the 2FA check on someone else's session if combined with another primary credential leak, or — in the worst implementations — the OTP step becomes meaningless entirely.

---

## 2. Missing 2FA Code Integrity Validation

**What it is:** Same root cause as #1, viewed from the validation-logic side: the backend's "is this code correct?" query doesn't filter `WHERE user_id = current_user`, so _any_ currently-valid code in the database satisfies the check.

**Steps:**

1. Request an OTP for your own throwaway/test account.
2. Attempt to use that exact code on the target session you're testing (a different account on the same app).
3. If accepted → **Critical**.

**Note:** This is functionally the same exploit as #1 — list both in reports only if the program treats them as distinct root causes (e.g. one is in the OTP table, one is in the session token table).

---

## 3. 2FA Bypass via OAuth / SSO Login Path

**What it is:** 2FA is enforced on the password-based login flow but the developers forgot to enforce it on alternative login methods (Google/Facebook/GitHub OAuth, SAML SSO, "Login with phone").

**Steps:**

1. Register/login normally with email + password.
2. Enable 2FA on the account.
3. Log out completely.
4. Log back in using the **OAuth/SSO** button instead of the password form (the OAuth account must be linked to the same email).
5. If you land in the dashboard **without ever being prompted for the OTP** → **Critical: 2FA bypass via OAuth**.

**Why it happens:** Developers treat OAuth as "already verified by a trusted third party" and skip the internal 2FA middleware check entirely for that code path.

---

## 4. 2FA Bypass via Password Reset Flow

**What it is:** The password-reset flow drops the user into an authenticated session (or auto-login) without ever checking the 2FA flag, because reset flows are often built as a separate, older code path that predates the 2FA feature.

**Steps:**

1. Log in normally, enable 2FA, log out.
2. Trigger **"Forgot Password"** and complete the reset via the emailed link.
3. Observe whether you land logged-in / auto-redirected to the dashboard **without** being asked for the OTP.
4. If yes → **Critical**.

**Related variant — "Password Reset Disables 2FA":** 5. After resetting (or just changing) the password while logged in, check the account's security settings. 6. If 2FA is now **silently disabled**, that's an even more severe variant — document both as separate findings if both occur.

---

##  5. Force Browsing Past the 2FA Step

**What it is:** The application treats "verified" as a client-side routing concept rather than a server-side session attribute. The server hands out a session cookie/token _before_ OTP verification succeeds, and authenticated endpoints don't check a `2fa_verified` flag — they only check "is there a valid session."

**Steps:**

1. Submit valid username/password. Do **not** submit the OTP.
2. With the session token/cookie you now hold, manually request sensitive authenticated endpoints in Burp Repeater:

```
   GET /dashboard
   GET /my-account
   GET /api/user/profile
   GET /api/settings
   GET /api/transactions
```

3. If any of these return real data instead of redirecting you back to `/2fa/verify` → **Critical: broken authentication state machine**.

**Tip:** This is the single highest-value test to run first on every new target — it's fast and frequently rewarding.

---

## 6. Response / Status Code Manipulation (Client-Side Trust)

**What it is:** The frontend decides whether to grant access based on a value in the HTTP response (a boolean flag or the status code) rather than the server enforcing access server-side on every subsequent request.

**Steps:**

1. Intercept the OTP verification response in Burp.
2. Submit a deliberately **wrong** OTP and intercept the response before it reaches the browser.
3. Modify the response body/status, trying each of:

```
   {"success": false}  →  {"success": true}
   {"verified": false} →  {"verified": true}
   0  →  1
   "failed" → "success"
```

```
   HTTP/1.1 403 Forbidden  →  HTTP/1.1 200 OK
   HTTP/1.1 401 Unauthorized → HTTP/1.1 200 OK
```

4. Forward the modified response and see if the frontend grants access.
5. If access is granted on a wrong OTP purely because of the modified response → **Critical**.

**Burp tip:** Use **Proxy → Match and Replace** rules to apply this automatically across an entire flow instead of manually intercepting every response.

---

# 🟠 HIGH

Strong impact, but usually requires an extra step, specific conditions, or doesn't grant _immediate_ full takeover on its own.

## 7. No Rate Limiting on OTP Verification (Brute Force)

**What it is:** A typical 4–6 digit numeric OTP has only 10,000–1,000,000 possible values. Without rate limiting or lockout, this is fully brute-forceable in minutes.

**Steps:**

1. Trigger the OTP step, intentionally don't enter the right code.
2. In Burp Intruder, set the OTP parameter as the payload position.
3. Use a **Numbers** payload set covering the full range (e.g. `000000`–`999999`, or the relevant length).
4. Launch the attack and filter responses by length / status code to spot the one that differs (the successful one).
5. If no lockout, CAPTCHA, or delay kicks in after dozens/hundreds of attempts → **High** (escalate to **Critical** if you can fully brute-force within a reasonable time window and demonstrate full account access).

**Tip:** Always demonstrate actual success (a valid session) in your PoC — a report that only shows "no lockout after N attempts" without proving the code was eventually found is weaker evidence.

---

## 8. Rate-Limit Bypass via Spoofed IP Headers

**What it is:** The app/WAF identifies the "client" for rate-limiting purposes using a header value instead of the real TCP source IP, because the app sits behind a reverse proxy/CDN and trusts forwarded headers blindly.

**Steps:**

1. Confirm rate limiting exists (e.g. blocked after 5 attempts from the same IP).
2. In Burp Intruder, brute-force the OTP again, but this time also vary one of the following headers on every request (use a different IP value each time, not just `127.0.0.1`):

  ```http
X-Originating-IP: 127.0.0.1
X-Forwarded-For: 127.0.0.1
X-Remote-IP: 127.0.0.1
X-Remote-Addr: 127.0.0.1
X-Forwarded-Host: 127.0.0.1
X-Client-IP: 127.0.0.1
X-Host: 127.0.0.1
Forwarded: for=127.0.0.1
X-Forwarded-By: 127.0.0.1
X-Forwarded-For-IP: 127.0.0.1
X-True-IP: 127.0.0.1
  ```
  
3. If the rate limit resets/never triggers because each request "appears" to come from a new IP → **High**.

**Tip:** Test headers one at a time first to identify exactly which one the backend trusts — this makes your report much more precise and useful to the triager.

---

## 9. Rate-Limit Bypass via Real IP Rotation

**What it is:** Same goal as #8, but bypasses servers that ignore spoofable headers entirely and check the real TCP-level source IP.

**Steps:**

1. Install the **Burp Suite extension "IP Rotate"** (BApp Store) — rotates real outbound IPs via AWS API Gateway.
2. Configure it and run the same OTP brute-force attack through Burp Intruder with IP rotation active.
3. If the lockout never triggers because every request truly originates from a different IP → **High**.

**Note:** Confirm the program's scope/rules allow this kind of traffic volume and AWS-based testing before running it against a live target.

---

## 10. CSRF on the "Disable 2FA" Action

**What it is:** The endpoint that turns off 2FA doesn't require a CSRF token and doesn't re-confirm identity (no password re-entry, no fresh OTP), so it can be triggered cross-site.

**Steps:**

1. Identify the disable-2FA request (e.g. `POST /account/2fa/disable`).
2. Check whether it requires: a CSRF token, the current password, or a fresh OTP. If none are required:
3. Build a minimal auto-submitting HTML PoC:

```html
   <form action="https://target.com/account/2fa/disable" method="POST" id="f">
   </form>
   <script>document.getElementById('f').submit();</script>
```

4. Host it and have a logged-in victim (or your own second test session) open it.
5. If 2FA is disabled on their account without any prompt → **High**.

---

## 11. Clickjacking on the "Disable 2FA" Page

**What it is:** The 2FA-disable page lacks `X-Frame-Options` / `Content-Security-Policy: frame-ancestors`, so it can be embedded in an invisible iframe and the victim tricked into clicking the real "Disable" button while believing they're interacting with attacker content.

**Steps:**

1. Check response headers on the disable-2FA page for `X-Frame-Options` or a CSP `frame-ancestors` directive.
2. If absent, build a PoC page that iframes the real page and overlays a fake "Click to continue" button positioned exactly over the real "Disable" button.
3. Social-engineer a test victim into clicking.
4. If 2FA gets disabled from the click → **High**.

---

## 12. Enabling 2FA Doesn't Invalidate Existing Sessions

**What it is:** If an attacker already hijacked a session (via XSS, token leak, etc.) _before_ the victim turned on 2FA, the victim's security upgrade doesn't help — the attacker's old session token remains fully valid.

**Steps:**

1. Using two browser profiles, log in as the same test account in both ("Attacker" profile and "Victim" profile).
2. In the Victim profile, enable 2FA (and/or change the password).
3. Go back to the Attacker profile's already-open session and refresh an authenticated page.
4. If the Attacker session is still active and not forced to re-authenticate → **High**.

---

## 13. Backup Code Abuse

**What it is:** Backup/recovery codes (the fallback when a user loses their authenticator device) frequently have weaker protections than the main OTP flow — they may be reusable, brute-forceable, or not user-bound.

**Steps:**

1. Generate backup codes for a test account.
2. Re-run the relevant tests from this sheet against the backup code endpoint specifically:
    - Reuse the same backup code twice (see #15).
    - Brute-force the backup code format if it's short/predictable.
    - Try using Account A's backup code against Account B's pending verification (see #1).
3. Any of these succeeding → **High** (escalate to Critical if it grants full account takeover with no further steps).

---

# 🟡 MEDIUM

Real findings worth reporting, but typically require specific conditions, give partial impact, or serve mainly as a stepping stone / DoS-style issue.

## 14. OTP/Token Exposed in HTTP Response

**What it is:** The endpoint that triggers OTP sending includes the actual code in the JSON response body (often a leftover debug field), or it leaks via response headers or verbose error messages.

**Steps:**

1. Trigger the "Send OTP" request and inspect the **full** raw response in Burp (not just what the UI displays) — check JSON fields, even oddly-named ones like `debug_otp`, `dev_code`, `_internal`.
2. Intentionally trigger backend errors (missing parameter, malformed request) and inspect those responses too.
3. Check response headers for anything unusual.
4. If the OTP value is present anywhere in a response reachable by the requesting client → **Medium** (escalate to **High/Critical** if this lets you fully take over _other_ users' accounts, e.g. if the leak isn't scoped to your own request).

---

## 15. Reusable OTP (Same Code Works Twice)

**What it is:** The backend never marks a code as "consumed" after a successful verification.

**Steps:**

1. Complete a normal login using a valid OTP.
2. Log out.
3. Start a new login attempt and submit the **exact same OTP** again.
4. If it's accepted again → **Medium**.

---

## 16. Previous OTP Doesn't Expire When a New One Is Issued

**What it is:** Requesting a new OTP should invalidate all earlier ones. If it doesn't, multiple codes are simultaneously valid.

**Steps:**

1. Request OTP #1 and note it (e.g. `123456`).
2. Without using it, request OTP #2 (e.g. `877678`).
3. Try submitting OTP #1.
4. If OTP #1 still works → **Medium**.

**Escalation trick:** If old codes stay valid indefinitely, request OTPs repeatedly (e.g. 20–50 times) to build up a _set_ of simultaneously-valid codes. This collapses the brute-force search space dramatically — instead of guessing 1 correct value out of 1,000,000, you may have, say, 50 correct values live at once, multiplying your odds in any subsequent brute-force/race attempt.

---

## 17. Bypass with Array Injection / Type Juggling

**What it is:** Some backends (PHP is the classic case, but Node/JS apps with loose validation are also vulnerable) don't strictly type-check the OTP parameter. If the code does something like a loose `==` comparison or loops over an array with `in_array()`, sending an **array of candidate codes** instead of a single string can make the check pass if _any_ element matches — or trigger type-juggling quirks (`null == false`, `0 == "abc"`, empty array oddities) that flip the result to true.

**Steps:**

1. Capture the OTP verification request body, e.g.:

```json
   { "otp": "123456" }
```

2. Change the value from a string to an array of multiple guesses:

```json
   {
     "otp": [
       "0000",
       "1111",
       "1234",
       "1337",
       "9999"
     ]
   }
```

3. Forward the request.
4. If the server accepts it as valid (because one element matched, or because of a type-confusion bug) → **Medium** (escalate based on actual impact — if this also bypasses rate-limiting because it's a single request testing many values, note that explicitly, since it compounds with #7/#8).

---

## 18. Null / Empty / Default-Value Bypass

**What it is:** Leftover testing/debug logic, or a backend that fails open when the comparison value is missing/null instead of failing closed.

**Steps:**

1. Intercept the OTP verification request.
2. Try replacing the OTP value with each of:

```
   ""
   null
   undefined
   " "
   "000000"
```

3. Forward each variant separately.
4. If any of these authenticates successfully → **Medium**.

---

## 19. JS File Analysis for Leaked OTP Logic

**What it is:** Rare, but worth checking — bundled JavaScript sometimes contains hardcoded test/debug values, OTP generation logic, or even cached codes from a previous response left in a global variable.

**Steps:**

1. Pull all JS files loaded during the auth flow (browser DevTools → Sources, or tools like `LinkFinder`/`getJS`).
2. Search for keywords: `otp`, `2fa`, `mfa`, `verify`, `code`, `debug`.
3. Check for hardcoded bypass values or test-mode flags (`if (env === 'test') skip2FA = true`).
4. Document anything sensitive found.

---

## 20. No Rate Limit on _Sending_ OTP (Resource/Financial Abuse)

**What it is:** Unlike #7 (brute-forcing the _verification_ endpoint), this targets the _send_ endpoint. Without a cooldown, an attacker can trigger unlimited SMS/email sends, racking up real cost on the company's SMS gateway (Twilio, etc.) or exhausting their provider quota — a Denial-of-Service / financial-impact issue rather than an auth bypass.

**Steps:**

1. Trigger "Send OTP" / "Resend Code" repeatedly.
2. Vary IP/session/headers (same techniques as #8/#9) if a per-IP cooldown exists.
3. Observe whether there's any cap, cooldown, or CAPTCHA after repeated triggers.

⚠️ **Ethical/safety note — read before testing this one specifically:**

- Only send the **minimum number of requests needed to prove the lack of rate limiting** (often 10–20 is plenty of evidence — you don't need thousands).
- Never run this against a real target without explicit scope permission for this _type_ of test; some programs exclude SMS-cost-related DoS from scope entirely. Causing genuine financial cost or exhausting a real SMS quota for legitimate users can cross from "bug bounty" into actual harm and potential legal/program exposure even under an otherwise-legitimate program.
- Stop immediately once you have a clear PoC; don't continue "for fun" or to see how high you can push it.

---

## 21. Enabling 2FA Without Email Verification (Pre-Account-Takeover Setup)

**What it is:** If a user can enable 2FA on an account whose email was never verified, an attacker who registers with someone else's unverified email-adjacent address (or exploits an email-confirmation bypass elsewhere) could lock the real owner out before they ever claim the account.

**Steps:**

1. Register a new account with an email you control but **do not** click the verification link.
2. Attempt to enable 2FA from account settings while the email is still unverified.
3. If the platform allows it → **Medium** (this is mainly a building block for a larger pre-account-takeover chain; rate it based on how it combines with other issues like email-claim flaws on that specific target).

---

## ✅ Suggested Testing Order (Fastest Wins First)

1. Force browsing (#5) — fast, often immediately rewarding
2. Response/status manipulation (#6) — fast
3. Cross-account OTP (#1) and integrity validation (#2) — needs 2 test accounts, high payoff
4. OAuth/SSO path (#3) and password-reset path (#4) — check every login method separately
5. Reusability/expiry checks (#15, #16) — quick, sets up brute-force escalation
6. Brute-force + rate-limit bypass (#7, #8, #9) — more time-intensive
7. Session/CSRF/clickjacking on account-security pages (#10, #11, #12, #13)
8. Leak hunting (#14, #19) and edge-case payloads (#17, #18)
9. Resource-abuse test (#20) — last, minimal requests, only if scope explicitly allows
