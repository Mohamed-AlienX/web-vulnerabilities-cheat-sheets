




# 🔐 Broken Session Management —  Cheat Sheet

> **Step-by-Step Testing Guide | OWASP WSTG Aligned | Severity-Ordered**

---

## 📌 What is Broken Session Management?

Broken Session Management occurs when an application **fails to properly invalidate, rotate, or protect sessions and tokens** after sensitive actions. This allows attackers to reuse old sessions, hijack accounts, or bypass authentication entirely.

**Sensitive actions that MUST trigger session/token invalidation:**
- Password change
- Logout
- Password reset
- Email verification
- Enabling 2FA / MFA
- Role/privilege escalation
- Account deletion

---

# 🔴 CRITICAL SEVERITY

---

## C1 — Password Reset Token Still Valid After Password Change

### 🎯 What It Is
The application fails to invalidate password reset tokens after the user changes their password through normal login. An attacker who previously requested a reset (or has access to old email) can still take over the account even after the legitimate user "secured" it.

### 🧪 Step-by-Step Testing

| Step | Action | Details |
|------|--------|---------|
| 1 | Request password reset | Submit reset request for your test account |
| 2 | Capture the reset link | Copy the token/link from email (DO NOT click it) |
| 3 | Login normally | Use current credentials to access the account |
| 4 | Change your password | Update password through normal settings |
| 5 | Logout | Fully terminate the session |
| 6 | Use the old reset link | Open the captured token in a new browser/session |
| 7 | Observe | Does the reset form still load? Can you set a new password? |

### ✅ Vulnerable Behavior
- Reset link still works and allows password change
- Token was NOT invalidated after password change

### ❌ Expected Behavior
- Reset link returns error: "Token invalid or expired"
- All outstanding reset tokens should be revoked on password change

### 💥 Impact
- **Full Account Takeover** — Attacker can reset password even after victim changes it
- Complete bypass of victim's security effort

### 📊 Severity: **CRITICAL / P1–P2**

---

## C2 — Session Fixation (Pre-Auth Session Becomes Authenticated)

### 🎯 What It Is
The application preserves the same session cookie value before and after authentication. An attacker can force a known session ID on a victim, wait for them to login, then use that same ID to hijack their authenticated session.

### 🧪 Step-by-Step Testing

| Step | Action | Details |
|------|--------|---------|
| 1 | Visit login page (no auth) | Capture the pre-auth session cookie (e.g., `JSESSIONID`) |
| 2 | Note the session ID value | Save it: `SESSION_ID_PRE = xxx` |
| 3 | Login with valid credentials | Submit username/password |
| 4 | Check post-auth session ID | Compare with pre-auth value |
| 5 | Test with forced cookie | In a new browser, set the PRE-auth cookie and visit authenticated page |
| 6 | Test concurrent sessions | Login as User A, logout, login as User B — check if same session ID is reused |

### ✅ Vulnerable Behavior
- Session ID does NOT change after login
- Pre-auth cookie grants authenticated access after victim logs in
- Same session ID reused across different user logins

### ❌ Expected Behavior
- New cryptographically random session ID issued after every successful authentication
- Old pre-auth session completely invalidated

### 💥 Impact
- **Account Takeover via Social Engineering** — Attacker tricks victim into using attacker-controlled session
- Bypasses authentication entirely

### 📊 Severity: **CRITICAL / P1–P2**

---

## C3 — Old Session Still Valid After Password Change (Cross-Device)

### 🎯 What It Is
When a user changes their password, the application fails to invalidate all active sessions across all devices/browsers. An attacker with a stolen session cookie retains access even after the victim changes their password.

### 🧪 Step-by-Step Testing

| Step | Action | Details |
|------|--------|---------|
| 1 | Login on Browser A | Chrome — note the session cookie |
| 2 | Login on Browser B | Firefox — same account, different session |
| 3 | Change password on Browser A | Use account settings to update password |
| 4 | Refresh Browser B | Press F5 or navigate to authenticated page |
| 5 | Test sensitive actions | Try updating profile, viewing private data on Browser B |
| 6 | Check session list (if available) | Some apps show active sessions — verify if old ones are killed |

### ✅ Vulnerable Behavior
- Browser B remains logged in after password change
- Can perform authenticated actions without re-entering new password

### ❌ Expected Behavior
- ALL sessions invalidated immediately upon password change
- All devices prompted to re-authenticate with new password

### 💥 Impact
- **Persistent Account Takeover** — Stolen session survives password change
- Victim's security action is completely ineffective

### 📊 Severity: **CRITICAL / P1–P2**

---

## C4 — Concurrent Session Handling Bypass (No Session Limit)

### 🎯 What It Is
The application allows unlimited concurrent sessions for the same user account without any restriction, notification, or invalidation of older sessions. This enables session hijacking to go undetected and allows attackers to maintain persistent access.

### 🧪 Step-by-Step Testing

| Step | Action | Details |
|------|--------|---------|
| 1 | Login from Device 1 | Browser A — capture session token |
| 2 | Login from Device 2 | Browser B — capture session token |
| 3 | Login from Device 3 | Browser C — capture session token |
| 4 | Check if Device 1 still works | Refresh page, perform action |
| 5 | Check session management UI | Does the app show active sessions? Can you revoke them? |
| 6 | Test cross-device replay | Copy Device 1 cookie to Device 4 — does it work? |
| 7 | Logout from Device 2 | Check if Device 1 and 3 are still active |

### ✅ Vulnerable Behavior
- Unlimited concurrent sessions allowed
- No notification to user about new logins
- No option to view/revoke active sessions
- Logout from one device doesn't affect others

### ❌ Expected Behavior
- Configurable concurrent session limit (e.g., max 3)
- Email/SMS notification on new device login
- User can view and terminate active sessions
- Option to "Log out all other devices" on password change

### 💥 Impact
- **Undetected Persistent Access** — Attacker maintains session indefinitely
- No way for victim to know they're compromised
- Credential change does NOT evict attacker

### 📊 Severity: **CRITICAL / P1–P2**

---

# 🔴 HIGH SEVERITY

---

## H1 — Password Reset Token Does Not Expire

### 🎯 What It Is
Password reset tokens remain valid for an excessively long time (hours, days, or indefinitely) instead of expiring after a short window (e.g., 15–30 minutes). If an attacker gains access to a victim's email (even old/archived), they can use stale reset links.

### 🧪 Step-by-Step Testing

| Step | Action | Details |
|------|--------|---------|
| 1 | Request password reset | Submit reset request |
| 2 | Capture the reset link | Save the full URL with token |
| 3 | Wait | Leave it untouched for 1 hour, 6 hours, 24 hours, 48 hours |
| 4 | Test at intervals | Try the link at each time checkpoint |
| 5 | Note expiration point | Record when the token finally fails |
| 6 | Test after use | Use token once, then try again — is it invalidated? |

### ✅ Vulnerable Behavior
- Token still works after 24+ hours
- Token works multiple times (not one-time use)
- No expiration timestamp in the token or server-side

### ❌ Expected Behavior
- Token expires after 15–30 minutes maximum
- Token is single-use (invalidated after first successful reset)
- Server-side tracking of issued tokens with expiration

### 💥 Impact
- **Account Takeover via Email Compromise** — Old emails with reset links become permanent backdoors
- Extended attack window for social engineering

### 📊 Severity: **HIGH / P2–P3**

---

## H2 — Session Still Valid After Logout (Server-Side Not Invalidated)

### 🎯 What It Is
The application clears the session cookie from the browser but fails to invalidate the session on the server. An attacker who captured the session token before logout can replay it to regain full access.

### 🧪 Step-by-Step Testing

| Step | Action | Details |
|------|--------|---------|
| 1 | Login to your account | Establish authenticated session |
| 2 | Intercept a sensitive request | Use Burp Proxy to capture a request (e.g., GET /profile) |
| 3 | Send to Repeater | Save the request with full headers and cookies |
| 4 | Logout normally | Click the logout button in the browser |
| 5 | Replay the request | In Burp Repeater, send the saved request again |
| 6 | Check response | Does it return 200 OK with user data? |
| 7 | Test with cookie restoration | Clear cookies, manually set the old session cookie, refresh |

### ✅ Vulnerable Behavior
- Request returns 200 OK with sensitive data after logout
- Old session cookie still grants authenticated access
- Server has no record of session invalidation

### ❌ Expected Behavior
- Server returns 401 Unauthorized or 403 Forbidden
- Session token removed from server-side store (Redis/DB)
- Old cookie value rejected even if manually restored

### 💥 Impact
- **Session Replay Attack** — Stolen session survives logout
- Shared/public device risk (cybercafé, library, office)

### 📊 Severity: **HIGH / P2–P3**

---

## H3 — 2FA / MFA Bypass via Session Reuse

### 🎯 What It Is
After enabling or verifying 2FA/MFA, the application does not invalidate pre-2FA sessions or does not require re-authentication. An attacker with a pre-2FA session can bypass the newly added security layer.

### 🧪 Step-by-Step Testing

| Step | Action | Details |
|------|--------|---------|
| 1 | Login without 2FA | Establish a session on an account without 2FA enabled |
| 2 | Capture the session cookie | Save the pre-2FA session token |
| 3 | Enable 2FA in settings | Set up TOTP or SMS 2FA on the account |
| 4 | Use the old cookie | In a new browser/incognito, set the pre-2FA cookie |
| 5 | Access protected resources | Try to access sensitive pages without entering 2FA code |
| 6 | Test session rotation | Check if session ID changes after 2FA verification |
| 7 | Test after 2FA disable | Disable 2FA, does the app require re-auth? |

### ✅ Vulnerable Behavior
- Pre-2FA session still works after 2FA is enabled
- No re-authentication required after enabling 2FA
- Session ID does not rotate after 2FA verification

### ❌ Expected Behavior
- All existing sessions invalidated when 2FA is enabled
- New session issued only after full 2FA verification
- Re-authentication required for sensitive settings changes

### 💥 Impact
- **2FA Bypass** — Attacker bypasses multi-factor authentication entirely
- Security control becomes ineffective

### 📊 Severity: **HIGH / P2–P3**

---

## H4 — Session Hijacking via Predictable/Weak Session IDs

### 🎯 What It Is
Session IDs are generated using weak algorithms (incremental numbers, timestamps, MD5 of username) or lack sufficient entropy. An attacker can predict or brute-force valid session IDs to hijack active sessions.

### 🧪 Step-by-Step Testing

| Step | Action | Details |
|------|--------|---------|
| 1 | Collect session samples | Login/logout multiple times, capture 20+ session IDs |
| 2 | Analyze structure | Check length, character set, patterns (hex, base64, numeric) |
| 3 | Test for encoding | Try base64 decode, hex decode, URL decode on the token |
| 4 | Check for information leakage | Does it contain username, timestamp, or IP address? |
| 5 | Test predictability | Compare sequential logins — is there a pattern? |
| 6 | Run entropy analysis | Use Burp Sequencer to test randomness |
| 7 | Attempt brute-force | Try modifying last few characters of a valid session ID |

### ✅ Vulnerable Behavior
- Session ID is short (< 128 bits / 32 hex chars)
- Contains user-identifiable information (username, email)
- Incremental or time-based patterns detected
- Low entropy score in Burp Sequencer

### ❌ Expected Behavior
- 128+ bits of entropy (32+ hex characters or equivalent)
- Cryptographically random generation (CSPRNG)
- No embedded user data or predictable structure
- Resistant to statistical analysis

### 💥 Impact
- **Mass Session Hijacking** — Attacker can enumerate and take over random user sessions
- Scalable attack without targeting specific victims

### 📊 Severity: **HIGH / P2–P3**

---

## H5 — Cookie Attributes Missing or Misconfigured

### 🎯 What It Is
Session cookies lack essential security attributes (`HttpOnly`, `Secure`, `SameSite`) or have overly broad `Domain`/`Path` scopes. This exposes sessions to XSS, MITM, and CSRF attacks.

### 🧪 Step-by-Step Testing

| Step | Action | Details |
|------|--------|---------|
| 1 | Login and capture response | Check `Set-Cookie` headers in HTTP response |
| 2 | Verify Secure flag | Is `Secure` present? Test over HTTP — is cookie sent? |
| 3 | Verify HttpOnly flag | Is `HttpOnly` present? Try `document.cookie` in console |
| 4 | Verify SameSite attribute | Check if `SameSite=Strict` or `SameSite=Lax` is set |
| 5 | Check Domain scope | Is it too broad (e.g., `.example.com` instead of `app.example.com`)? |
| 6 | Check Path scope | Is it `/` when it should be more restrictive? |
| 7 | Test cookie prefix | Are `__Host-` or `__Secure-` prefixes used? |
| 8 | Test over HTTP | Change HTTPS to HTTP — does the cookie still transmit? |

### ✅ Vulnerable Behavior
- Missing `HttpOnly` → XSS can steal cookies via JavaScript
- Missing `Secure` → Cookie sent over unencrypted HTTP
- Missing `SameSite` → Vulnerable to CSRF attacks
- Overly broad `Domain` → Cookie accessible to subdomains
- No `__Host-` or `__Secure-` prefix

### ❌ Expected Behavior
```
Set-Cookie: __Host-SID=<token>; Path=/; Secure; HttpOnly; SameSite=Strict
```

### 💥 Impact
- **XSS → Session Theft** (if no HttpOnly)
- **MITM → Session Interception** (if no Secure)
- **CSRF → Unauthorized Actions** (if no SameSite)

### 📊 Severity: **HIGH / P2–P3**

---

## H6 — Exposed Session Variables (Session ID in URL/Logs)

### 🎯 What It Is
Session identifiers are transmitted in URLs (GET parameters), page bodies, or referrer headers. This exposes them in browser history, proxy logs, server logs, and referrer headers sent to third parties.

### 🧪 Step-by-Step Testing

| Step | Action | Details |
|------|--------|---------|
| 1 | Navigate the application | Watch URL bar for session IDs |
| 2 | Check all links | Hover over links — any `?sessionid=xxx` or `?sid=xxx`? |
| 3 | Check page source | Search for session tokens in HTML/JS |
| 4 | Check referrer behavior | Click external link, check if session ID leaks in Referer header |
| 5 | Check server logs | If you have access, check if session IDs appear in access logs |
| 6 | Test POST vs GET | Can POST requests with session data be converted to GET? |
| 7 | Check for URL rewriting | Does the app rewrite URLs to include session ID? |

### ✅ Vulnerable Behavior
- `JSESSIONID` or similar appears in URL
- Session token visible in page source
- Session ID leaked in Referer to third-party sites
- Server logs contain session identifiers

### ❌ Expected Behavior
- Session ID transmitted ONLY in cookies (or secure headers)
- No session data in URLs, page content, or logs
- POST-only for sensitive operations

### 💥 Impact
- **Session Leakage via Logs/History** — Anyone with log access can hijack sessions
- **Referrer Header Leakage** — Third parties receive session IDs
- **Browser History Exposure** — Shared computer risk

### 📊 Severity: **HIGH / P2–P3**

---

# 🟠 MEDIUM SEVERITY

---

## M1 — Email Verification Bypass (Change Email Trick)

### 🎯 What It Is
The application fails to reset verification status when a user changes their email address. An attacker can verify a temporary email, then change back to an unverified email that inherits the verified status.

### 🧪 Step-by-Step Testing

| Step | Action | Details |
|------|--------|---------|
| 1 | Create account with Email A | Use an email you control (do NOT verify) |
| 2 | Change email to Email B | Update email in account settings |
| 3 | Verify Email B | Click the verification link sent to Email B |
| 4 | Change email back to Email A | Switch back to the original unverified email |
| 5 | Check verification status | Does Email A now show as "verified"? |
| 6 | Test account restrictions | Can you access features that require verified email? |

### ✅ Vulnerable Behavior
- Email A appears verified despite never being verified
- Account gains verified privileges without actual email ownership

### ❌ Expected Behavior
- Changing email resets verification status to "unverified"
- New email requires fresh verification
- Old verified status does NOT transfer

### 💥 Impact
- **Bypass signup verification requirements**
- **Business logic abuse** — Access features requiring verified email without owning it
- Can be chained with other attacks (password reset to unverified email)

### 📊 Severity: **MEDIUM / P3–P4**

---

## M2 — Verification Applied to Wrong Email

### 🎯 What It Is
When a user changes their email and clicks the verification link, the application verifies the OLD email instead of the new one. This allows an attacker to verify an email they don't own.

### 🧪 Step-by-Step Testing

| Step | Action | Details |
|------|--------|---------|
| 1 | Create account with victim@email.com | Use victim's email (you don't control) |
| 2 | Change email to attacker@email.com | Update to your controlled email |
| 3 | Click verification link | The link sent to attacker@email.com |
| 4 | Check which email is verified | Query the account — which email shows as verified? |
| 5 | Test with API | Check `/api/user` or profile endpoint for verified_email field |

### ✅ Vulnerable Behavior
- victim@email.com becomes verified instead of attacker@email.com
- Attacker gains verified status on email they don't own

### ❌ Expected Behavior
- Only the email that received the verification link becomes verified
- Verification is bound to the specific email address, not the account generically

### 💥 Impact
- **Email ownership bypass** — Attacker verifies victim's email without access
- Can enable password reset to victim's email

### 📊 Severity: **MEDIUM / P3**

---

## M3 — Replay After Logout (Authenticated Request Still Valid)

### 🎯 What It Is
After logout, previously intercepted authenticated requests can still be replayed successfully. The server fails to recognize that the session has been terminated.

### 🧪 Step-by-Step Testing

| Step | Action | Details |
|------|--------|---------|
| 1 | Login to your account | Establish authenticated session |
| 2 | Perform a sensitive action | Update profile, change settings, transfer funds |
| 3 | Intercept the request | Use Burp Proxy to capture the full request |
| 4 | Send to Repeater | Save the request in Burp Repeater |
| 5 | Logout from your account | Click logout button in browser |
| 6 | Replay the request | Send the saved request from Repeater |
| 7 | Observe response | Does it return 200 OK? Was the action performed? |

### ✅ Vulnerable Behavior
- Request returns 200 OK after logout
- Action is successfully performed (e.g., profile updated)
- No re-authentication challenge presented

### ❌ Expected Behavior
- Server returns 401 Unauthorized or 403 Forbidden
- Session token invalidated server-side
- Action is rejected

### 💥 Impact
- **Session reuse after logout** — Attacker can replay stolen requests
- Account actions without authentication on shared devices

### 📊 Severity: **MEDIUM / P3–P4**

---

## M4 — Session Timeout Not Enforced (Idle Timeout Bypass)

### 🎯 What It Is
The application does not enforce an idle session timeout, or the timeout is controlled client-side (e.g., via cookie expiration). Sessions remain valid indefinitely, increasing the window for hijacking.

### 🧪 Step-by-Step Testing

| Step | Action | Details |
|------|--------|---------|
| 1 | Login and capture session | Note the session cookie and current time |
| 2 | Leave session idle | Do NOT interact with the application for 30+ minutes |
| 3 | Test after 30 min | Send a request with the old cookie |
| 4 | Test after 1 hour | Send another request |
| 5 | Test after 2+ hours | Continue testing until session fails |
| 6 | Check cookie expiration | Is there an `Expires` or `Max-Age` attribute? |
| 7 | Modify cookie time | If client-side time tracking, try manipulating cookie timestamps |

### ✅ Vulnerable Behavior
- Session still works after hours of inactivity
- No server-side timeout enforcement
- Cookie contains client-manipulable time data

### ❌ Expected Behavior
- Session expires after 15–30 minutes of inactivity (banking: 5–15 min)
- Timeout enforced server-side (not client-side)
- Automatic redirect to login page after timeout

### 💥 Impact
- **Extended attack window** — Stolen sessions remain valid longer
- Public/shared computer risk

### 📊 Severity: **MEDIUM / P3–P4**

---

## M5 — Session Puzzling (Variable Overloading)

### 🎯 What It Is
The application uses the same session variable for multiple purposes (e.g., password reset flow AND authentication check). An attacker can set the variable in one context (public page) and exploit it in another (protected page) to bypass authentication.

### 🧪 Step-by-Step Testing

| Step | Action | Details |
|------|--------|---------|
| 1 | Map all session variables | Identify every session variable the app uses |
| 2 | Identify entry points | Find public pages that set session variables (password reset, guest checkout) |
| 3 | Identify exit points | Find protected pages that check session variables for auth |
| 4 | Test sequence A | Visit password reset page → enter email → don't complete → visit admin page |
| 5 | Test sequence B | Add item to cart (as guest) → visit checkout → check if auth bypassed |
| 6 | Test sequence C | Complete step 1 of multi-step form → jump to step 3 directly |
| 7 | Check variable overlap | Does `session['user_id']` get set by both auth AND password reset? |

### ✅ Vulnerable Behavior
- Authentication bypassed by visiting public page first
- Multi-step process skipped by setting variables out of order
- Privilege escalation via variable set in low-privilege context

### ❌ Expected Behavior
- Each session variable has a single, well-defined purpose
- Authentication variables ONLY set after proper credential verification
- Multi-step flows validate completion of each step

### 💥 Impact
- **Authentication Bypass** — Access protected areas without credentials
- **Privilege Escalation** — Gain admin access from user session
- **Business Logic Abuse** — Skip payment/verification steps

### 📊 Severity: **MEDIUM / P3–P4**

---

## M6 — Old Password Reset Token Valid After New Request

### 🎯 What It Is
When a user requests multiple password resets, the older tokens remain valid alongside newer ones. An attacker with access to any reset email can use any token, not just the latest.

### 🧪 Step-by-Step Testing

| Step | Action | Details |
|------|--------|---------|
| 1 | Request reset → Token 1 | Capture first reset link |
| 2 | Wait 5 minutes | Allow time gap |
| 3 | Request reset again → Token 2 | Capture second reset link |
| 4 | Use Token 1 | Try the first (older) reset link |
| 5 | Use Token 2 | Try the second (newer) reset link |
| 6 | Check after Token 2 use | Is Token 1 still valid after Token 2 is consumed? |

### ✅ Vulnerable Behavior
- Token 1 still works after Token 2 is issued
- Multiple valid tokens exist simultaneously
- No single-use or chain invalidation

### ❌ Expected Behavior
- Only the most recent token is valid
- Using any token invalidates all other tokens for that account
- Maximum ONE valid reset token per account at any time

### 💥 Impact
- **Token Replay** — Old emails with reset links remain exploitable
- Increased attack surface with multiple valid tokens

### 📊 Severity: **MEDIUM / P3–P4**

---

# 🟡 LOW SEVERITY

---

## L1 — Cached Pages After Logout (Back Button Issue)

### 🎯 What It Is
After logout, pressing the browser back button reveals cached authenticated pages with sensitive data. While the data is from cache (not live), it still exposes information on shared devices.

### 🧪 Step-by-Step Testing

| Step | Action | Details |
|------|--------|---------|
| 1 | Login to your account | Access sensitive pages (profile, dashboard) |
| 2 | Logout | Click the logout button |
| 3 | Press Alt + Left Arrow | Or click browser back button multiple times |
| 4 | Check page content | Do you see profile data, PII, or sensitive info? |
| 5 | Press F5 / Refresh | Does the page reload from server or show cache? |
| 6 | Check response headers | Look for `Cache-Control`, `Pragma`, `Expires` headers |

### ✅ Vulnerable Behavior
- Sensitive data visible after logout via back button
- No cache-control headers preventing storage
- Page renders from browser cache without server check

### ❌ Expected Behavior
- `Cache-Control: no-store, no-cache, must-revalidate, private`
- `Pragma: no-cache`
- `Expires: 0`
- Immediate redirect to login on back navigation

### 💥 Impact
- **Sensitive data exposure** on shared/public devices
- **Privacy risk** — Next user can see previous user's data

### 📊 Severity: **LOW / P4–P5**

---

## L2 — Logout UI/UX Issues (Hidden or Ambiguous Logout)

### 🎯 What It Is
The logout function is hidden, ambiguous, or missing entirely. Users may not trust or find the logout feature, leading to sessions remaining active longer than intended.

### 🧪 Step-by-Step Testing

| Step | Action | Details |
|------|--------|---------|
| 1 | Check every page | Is a logout button/link visible on ALL authenticated pages? |
| 2 | Check mobile view | Is logout accessible on mobile/responsive design? |
| 3 | Check scroll requirement | Do you need to scroll to find logout? |
| 4 | Check naming | Is it called "Logout", "Sign Out", or something ambiguous? |
| 5 | Check placement | Is it in a standard location (top-right)? |
| 6 | Test without clicking | Close browser tab — does session persist? |

### ✅ Vulnerable Behavior
- No logout button on some pages
- Logout hidden in menus or requires scrolling
- Ambiguous naming (e.g., "Exit" instead of "Logout")
- No automatic session termination on browser close

### ❌ Expected Behavior
- Logout visible on ALL pages without scrolling
- Clear, unambiguous labeling
- Standard placement (top navigation bar)
- Session timeout even if user doesn't manually logout

### 💥 Impact
- **User behavior risk** — Users leave sessions active
- Increased window for session hijacking

### 📊 Severity: **LOW / P4–P5**

---

## L3 — SSO Session Not Terminated Globally

### 🎯 What It Is
In Single Sign-On (SSO) environments, logging out of one application does not terminate sessions in connected applications or the SSO provider itself. Users can re-access applications without re-authentication.

### 🧪 Step-by-Step Testing

| Step | Action | Details |
|------|--------|---------|
| 1 | Login via SSO | Access the app through SSO provider (e.g., Okta, Azure AD) |
| 2 | Access multiple apps | Navigate to other apps in the same SSO ecosystem |
| 3 | Logout from App A | Use App A's logout function |
| 4 | Check App B | Is App B still accessible without re-auth? |
| 5 | Check SSO portal | Can you re-enter App A from SSO portal without password? |
| 6 | Logout from SSO portal | Use central SSO logout |
| 7 | Check all apps | Are ALL app sessions terminated? |

### ✅ Vulnerable Behavior
- App B remains accessible after logging out of App A
- SSO portal allows re-entry without re-authentication
- Individual app logout doesn't trigger global session termination

### ❌ Expected Behavior
- Logout from ANY app triggers global session termination
- SSO portal requires re-authentication after any app logout
- All connected applications reject the session token

### 💥 Impact
- **Incomplete session termination** — User thinks they're logged out everywhere
- Re-authentication bypass across SSO ecosystem

### 📊 Severity: **LOW / P4–P5**

---

## L4 — Session ID Not Rotated After Privilege Change

### 🎯 What It Is
When a user's privileges change (e.g., promoted to admin, role assignment), the session ID remains the same. An attacker with the old session can access new privileges without re-authentication.

### 🧪 Step-by-Step Testing

| Step | Action | Details |
|------|--------|---------|
| 1 | Login as regular user | Capture session cookie |
| 2 | Note session ID | Save the token value |
| 3 | Elevate privileges | Have admin promote your account to admin role |
| 4 | Check session ID | Did it change after privilege escalation? |
| 5 | Test old cookie | In a new browser, set the pre-elevation cookie |
| 6 | Access admin functions | Try admin-only endpoints with old session |

### ✅ Vulnerable Behavior
- Session ID unchanged after role change
- Old session grants new privileges
- No re-authentication required for privilege escalation

### ❌ Expected Behavior
- Session invalidated on ANY privilege change
- New session issued after re-authentication
- Old session cannot access elevated privileges

### 💥 Impact
- **Privilege Escalation via Old Session** — Attacker gains admin access with user session

### 📊 Severity: **LOW / P4–P5**

---

# 📋 Quick Reference Table

| ID | Vulnerability | Severity | OWASP WSTG | Key Test |
|----|-------------|----------|------------|----------|
| C1 | Reset Token Valid After Password Change | 🔴 Critical | WSTG-SESS-01 | Request reset → change pass → use old token |
| C2 | Session Fixation | 🔴 Critical | WSTG-SESS-03 | Pre-auth cookie = post-auth cookie |
| C3 | Old Session Valid After Password Change | 🔴 Critical | WSTG-SESS-06 | Change pass on Browser A → check Browser B |
| C4 | Concurrent Session Bypass | 🔴 Critical | WSTG-SESS-06 | Login from 5+ devices simultaneously |
| H1 | Reset Token Never Expires | 🔴 High | WSTG-SESS-01 | Wait 24h+ → test token |
| H2 | Session Valid After Logout | 🔴 High | WSTG-SESS-06 | Intercept → logout → replay |
| H3 | 2FA Bypass via Session Reuse | 🔴 High | WSTG-SESS-03 | Pre-2FA cookie after 2FA enable |
| H4 | Predictable Session IDs | 🔴 High | WSTG-SESS-01 | Burp Sequencer entropy test |
| H5 | Missing Cookie Attributes | 🔴 High | WSTG-SESS-02 | Check Secure/HttpOnly/SameSite |
| H6 | Session ID in URL/Logs | 🔴 High | WSTG-SESS-04 | Check URLs, source, referrers |
| M1 | Email Verification Bypass | 🟠 Medium | — | Change email trick |
| M2 | Wrong Email Verification | 🟠 Medium | — | Verify old email instead of new |
| M3 | Replay After Logout | 🟠 Medium | WSTG-SESS-06 | Repeater replay post-logout |
| M4 | No Session Timeout | 🟠 Medium | WSTG-SESS-07 | Wait 2h+ → test session |
| M5 | Session Puzzling | 🟠 Medium | WSTG-SESS-08 | Set variable in public → use in protected |
| M6 | Old Reset Token After New Request | 🟠 Medium | WSTG-SESS-01 | Token 1 valid after Token 2 issued |
| L1 | Cached Pages After Logout | 🟡 Low | WSTG-SESS-06 | Back button after logout |
| L2 | Hidden Logout UI | 🟡 Low | WSTG-SESS-06 | Check visibility on all pages |
| L3 | SSO Incomplete Logout | 🟡 Low | WSTG-SESS-06 | Cross-app session persistence |
| L4 | No Session Rotation on Privilege Change | 🟡 Low | WSTG-SESS-03 | Role change → same session ID |

---

# 📝 Report Writing Tips

1. **Always include timestamps** — Show before/after states clearly
2. **Include full HTTP requests/responses** — Redact only credentials
3. **Demonstrate impact** — Show what an attacker could do, not just that it "works"
4. **Chain vulnerabilities** — Show how Low + Medium = Critical
5. **Suggest remediation** — Don't just find, help fix
6. **Use video/GIF for complex flows** — Session management bugs are often flow-dependent

---

> **Remember:** Session management vulnerabilities are often about **what the app FAILS to do** (invalidate, rotate, expire) rather than what it actively does wrong. Always test the negative cases.
