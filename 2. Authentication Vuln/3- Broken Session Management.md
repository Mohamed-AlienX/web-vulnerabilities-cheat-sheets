


# Practical Cheat Sheet

_(Step-by-Step – Bug Bounty Mindset)_

---

## 🧠 What is Broken Session Management?

Broken Session Management happens when the application **fails to properly handle sessions or tokens** after sensitive actions like:

- Password change
    
- Logout
    
- Password reset
    
- Email verification
    
- Enabling 2FA
    

---

## 1️⃣ Old Session Still Valid After Password Change

### Steps to Test

1. Login to the same account on **Browser A** and **Browser B**
    
2. Change the password on **Browser A**
    
3. Refresh the page on **Browser B**
    

### Expected Behavior

❌ Browser B should be logged out

### Vulnerable Behavior

✅ Browser B is still logged in and can access the account

### Impact

- Attacker with stolen session cookie keeps access
    
- **Account Takeover**
    

### Severity

🔴 **High / P3**

---

## 2️⃣ Password Reset Token Still Valid After Password Change

### Steps to Test

1. Request a password reset
    
2. Receive the reset link (do NOT click it)
    
3. Login normally
    
4. Change your password
    
5. Logout
    
6. Use the old reset link
    

### Expected Behavior

❌ Reset link should be invalid

### Vulnerable Behavior

✅ Reset link still works

### Impact

- Full account takeover
    
- Critical logic failure
    

### Severity

🔴 **High / P3 – P2**

---

## 3️⃣ Password Reset Token Does Not Expire

### Steps to Test

1. Request password reset
    
2. Wait (or logout)
    
3. Use the reset link after a long time
    

### Expected Behavior

❌ Token expires after short time or after use

### Vulnerable Behavior

✅ Token still works

### Impact

- Account takeover if email is compromised
    

### Severity

🔴 **High / P3**

---

## 4️⃣ Email Verification Bypass (Change Email Trick)

### Steps to Test

1. Create account with **email A** (not verified)
    
2. Change email to **email B**
    
3. Verify email B
    
4. Change email back to **email A**
    

### Expected Behavior

❌ Email A should be unverified

### Vulnerable Behavior

✅ Email A appears verified

### Impact

- Bypass signup verification
    
- Business logic abuse
    

### Severity

🟠 **Medium / P3 – P4**

---

## 5️⃣ Verification Applied to Wrong Email

### Steps to Test

1. Create account with [victim@email.com](mailto:victim@email.com) (not verified)
    
2. Change email to [attacker@email.com](mailto:attacker@email.com)
    
3. Click verification link
    
4. Check which email is verified
    

### Expected Behavior

❌ Only [attacker@email.com](mailto:attacker@email.com) should be verified

### Vulnerable Behavior

✅ [victim@email.com](mailto:victim@email.com) becomes verified

### Impact

- Email ownership bypass
    

### Severity

🟠 **Medium / P3**

---

##  6️⃣ Replay After Logout

### Test Name

**Authenticated Request Still Valid After Logout**

### Steps

1. Login to your account
    
2. Perform any sensitive action  
    _(update profile / change settings / update data)_
    
3. Intercept the request using **Burp Suite**
    
4. Send the request to **Repeater**
    
5. Logout from your account
    
6. Replay the same request from **Repeater**
    

### Vulnerable If

- Request returns **200 OK**
    
- Action is successfully performed
    
- No re-authentication is required
    

### Expected Behavior

- Session should be invalid
    
- Server should return **401 / 403**
    

### Impact

- Session reuse after logout
    
- Account actions without authentication
    
- Possible account takeover on shared devices
    

### Severity

🟠 **Medium /(P3 / P4)**

### OWASP Category

**Broken Authentication and Session Management**  
**Failure to Invalidate Session**

---

## 7️⃣ Session Still Valid After Logout (Using Old Request)

### Steps to Test

1. Login and intercept a sensitive request (Burp)
    
2. Send request to Repeater
    
3. Logout from account
    
4. Replay the intercepted request
    

### Expected Behavior

❌ Request should fail (401 / 403)

### Vulnerable Behavior

✅ Request still succeeds

### Impact

- Session not invalidated
    
- Limited unauthorized actions
    

### Severity

🟡 **Low / P4**

---

## 8️⃣ Old Password Reset Token Still Valid After New Request

### Steps to Test

1. Request password reset → Token 1
    
2. Request password reset again → Token 2
    
3. Use Token 1
    

### Expected Behavior

❌ Token 1 should be invalid

### Vulnerable Behavior

✅ Token 1 still works

### Impact

- Depends on token lifetime
    
- Some programs may reject
    

### Severity

🟡 **Low / P4**

---

##  Cached Pages After Logout (Back Button Issue)

### Steps to Test

1. Login
    
2. Logout
    
3. Press **Alt + Left Arrow** (Back)
    
4. Check page content
    

### Expected Behavior

❌ Redirect to login page

### Vulnerable Behavior

✅ Profile data still visible

### Impact

- Sensitive data exposure
    
- Shared device risk
    

### Severity

🟡 **Low / P4**

---

