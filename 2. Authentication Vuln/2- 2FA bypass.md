

# 🔐 2FA / OTP Bypass Cheat Sheet

**(Checklist + Testing Steps)**

---

## 🧠 Mental Model (Quick Pre-Check)

Before testing any 2FA implementation, always ask:

- Is the OTP **bound to the user account**?
    
- Is the OTP **one-time only** or reusable?
    
- Is OTP verification **rate-limited**?
    
- Is 2FA enforced on **ALL authentication methods**?
    
- Are **all active sessions invalidated** after enabling 2FA or changing the password?
    

---

## 1️⃣ Weak / Default OTP Values

### ✅ Checklist

-  Test common OTP values
    
-  Test the **first OTP** generated
    
-  Test **staging / test environments**
    

### 🧪 Testing Steps

1. Login until you reach the OTP verification step
    
2. Intercept the OTP verification request (Burp / Proxy)
    
3. Try common values:
    
    ```
    000000
    111111
    123456
    654321
    ```
    
4. Observe:
    
    - HTTP status code
        
    - Response body
        
5. If login succeeds → ❌ **Weak OTP validation**
    

---

## 2️⃣ Null / Empty OTP

### ✅ Checklist

-  Empty value
    
-  null
    
-  undefined
    
-  Space character
    

### 🧪 Testing Steps

1. Intercept the OTP verification request
    
2. Replace the OTP value with:
    
    ```
    ""
    null
    undefined
    " "
    ```
    
3. Forward the request
    
4. If authentication succeeds → ❌ **Missing backend validation**
    

---

## 3️⃣ Reuse Previous OTP (Used OTP)

### ✅ Checklist

-  OTP reuse in same session
    
-  OTP reuse after logout
    
-  OTP reuse in new session
    

### 🧪 Testing Steps

1. Submit a valid OTP and complete login
    
2. Logout
    
3. Try submitting the **same OTP again**
    
4. If accepted → ❌ **OTP not invalidated**
    

---

## 4️⃣ OTP Reuse Across Accounts

### ✅ Checklist

-  OTP not bound to user
    
-  Cross-account OTP testing
    

### 🧪 Testing Steps

1. Account A:
    
    - Request OTP and capture it
        
2. Account B:
    
    - Attempt to verify OTP from Account A
        
3. If successful → ❌ **OTP not bound to user (Critical)**
    

---

## 5️⃣ No Rate Limit on OTP Verification

### ✅ Checklist

-  Unlimited OTP attempts
    
-  No temporary lockout
    
-  Rate limit bypassable
    

### 🧪 Testing Steps

1. Submit incorrect OTP multiple times (10–50 attempts)
    
2. Check for:
    
    - Account lock
        
    - CAPTCHA
        
    - Temporary block
        
3. Try bypassing by:
    
    - Changing IP
        
    - New session
        
    - New cookies
        
4. If unlimited → ❌ **OTP brute force possible**
    

---

## 6️⃣ OTP Exposed in Response

### ✅ Checklist

-  Response body
    
-  HTTP headers
    
-  Error messages
    

### 🧪 Testing Steps

1. Trigger backend errors intentionally:
    
    - Missing parameters
        
    - Invalid OTP
        
2. Inspect:
    
    - JSON response
        
    - Debug fields
        
3. If OTP is leaked → ❌ **Sensitive data exposure**
    

---

## 7️⃣ Bypass 2FA via Password Reset Flow

**Classic authentication logic flaw**

### ✅ Checklist

-  Password reset skips 2FA
    
-  Auto-login after reset
    

### 🧪 Testing Steps

1. Login normally
    
2. Enable 2FA
    
3. Logout
    
4. Click **Forgot Password**
    
5. Open password reset email link
    
6. If logged in **without OTP** → 🔥 **2FA Bypass**
    

---

## 8️⃣ Bypass 2FA via OAuth (Google / Facebook)

**Authentication flow inconsistency**

### ✅ Checklist

-  OAuth login tested
    
-  2FA not enforced on OAuth
    

### 🧪 Testing Steps

1. Login with username/password
    
2. Enable 2FA
    
3. Logout
    
4. Login using Google / Facebook OAuth
    
5. If OTP not required → 🔥 **2FA bypass via OAuth**
    

---

## 9️⃣ No Rate Limit on Sending OTP

### ✅ Checklist

-  Unlimited OTP sending
    
-  No cooldown
    
-  Rate limit bypassable
    

### 🧪 Testing Steps

1. Repeatedly click:
    
    - Send OTP
        
    - Resend Code
        
2. Send 20–30 requests
    
3. Change IP / session
    
4. If unlimited → ❌ **OTP flooding / DoS risk**
    

---

## 🔟 Response Manipulation (Client-Side Trust)

### ✅ Checklist

-  Frontend trusts backend response
    
-  No server-side enforcement
    

### 🧪 Testing Steps

1. Intercept server response
    
2. Modify values:
    
    ```
    403 → 200
    false → true
    0 → 1
    failed → success
    "verified": false → true
    ```
    
3. If access granted → 🔥 **Client-side validation flaw**
    

---

## 1️⃣1️⃣ Skip 2FA Step (Forced Browsing)

### ✅ Checklist

-  Direct URL access allowed
    
-  Missing OTP enforcement
    

### 🧪 Testing Steps

1. Login with username/password
    
2. Before OTP verification, manually visit:
    
|  ```
    /dashboard
    /profile
    /account
    ```
    
3. If accessible → ❌ **Broken authentication flow**
    

---

## 1️⃣2️⃣ Enable 2FA Without Email Verification

### ✅ Checklist

-  Email unverified
    
-  2FA enabled successfully
    

### 🧪 Testing Steps

1. Register a new account
    
2. Do NOT verify email
    
3. Enable 2FA
    
4. If allowed → 🔥 **Pre-Account Takeover**
    

---

## 1️⃣3️⃣ Enabling 2FA Does NOT Invalidate Other Sessions

### ✅ Checklist

-  Old sessions remain active
    
-  Password change ignored
    

### 🧪 Testing Steps

1. Attacker logs in and keeps session active
    
2. Victim enables 2FA or changes password
    
3. Test attacker session
    
4. If still valid → ❌ **Session fixation / persistence**
    

---



