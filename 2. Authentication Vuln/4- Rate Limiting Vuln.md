


# 🚦 Rate Limiting Cheat Sheet (Web)

## 🧠 What is Rate Limiting?

Rate limiting means the application **limits the number of requests** a user/IP can send in a specific time.

If there is **NO rate limit**, an attacker can:

- Bruteforce passwords
    
- Bypass OTP / 2FA
    
- Spam users
    
- Cause Denial of Service (DoS)
    

---

# 🔥 Impact Ranking (Most → Least Critical)

1. **Login / OTP / 2FA** → Account Takeover (Critical)
    
2. **Password reset (send link / OTP)** → Account Takeover
    
3. **Internal password check endpoint**
    
4. **Port 22 (SSH)** → Server compromise
    
5. **Reports / abuse systems**
    
6. **Comments**
    
7. **Contact Us**
    
8. **Spam-only features**
    

---

# 1️⃣ No Rate Limit on Login Page (🔥 Critical)

### 🎯 Goal

Check if you can brute-force passwords without getting blocked.

### 🧪 Steps

1. Go to `/login`
    
2. Intercept request with **Burp Suite**
    
3. Send request to **Intruder / ffuf**
    
4. Try many passwords quickly
    

```http
POST /login.php HTTP/1.1
Host: target.com

username=admin&password=FUZZ
```

### 🚨 Vulnerability

- No CAPTCHA
    
- No lockout
    
- No delay
    
- Same response every time
    

### 💥 Impact

➡️ **Account Takeover**

---

# 2️⃣ No Rate Limit on Internal Password Check

### 🎯 Goal

Some apps check password via API before login.

### 🧪 Steps

1. Look for API like:
    

```http
POST /api/check-password
```

2. Send many requests with different passwords
    
3. No blocking = vulnerable
    

### 💥 Impact

➡️ Stealth brute-force  
➡️ Easier than login brute-force

---

# 3️⃣ No Rate Limit on Sending Reset Password Link

### 🎯 Goal

Spam reset emails or abuse OTP logic.

### 🧪 Steps

1. Go to "Forgot Password"
    
2. Intercept request
    
3. Send request many times
    

```http
POST /reset-password
email=victim@target.com
```

### 🚨 Signs

- Unlimited emails
    
- Same response always
    
- No cooldown
    

### 💥 Impact

➡️ Account takeover  
➡️ Email bombing

---

# 4️⃣ No Rate Limit on OTP / 2FA (🔥🔥 Critical)

### 🎯 Goal

Brute-force OTP code.

### 🧪 Steps

1. Login with correct password
    
2. Reach OTP page
    
3. Intercept OTP request
    
4. Brute-force OTP (000000 → 999999)
    

```http
POST /verify-otp
otp=FUZZ
```

### 🚨 Vulnerability

- No attempt limit
    
- No lockout
    
- OTP reusable
    

### 💥 Impact

➡️ **Full Account Takeover**

---

# 5️⃣ No Rate Limit on Contact Us Page

### 🎯 Goal

Spam or DoS support system.

### 🧪 Steps

1. Intercept contact form
    
2. Send same request many times
    

### 💥 Impact

➡️ Spam  
➡️ Operational impact (Low–Medium)

---

# 6️⃣ No Rate Limit on Comments

### 🎯 Goal

Spam comments or abuse storage.

### 🧪 Steps

1. Post comment
    
2. Repeat fast with script
    

### 💥 Impact

➡️ Spam  
➡️ Storage abuse

---

# 7️⃣ No Rate Limit on Comment Reports

### 🎯 Goal

Abuse moderation system.

### 🧪 Steps

1. Report same comment many times
    
2. No blocking or delay
    

### 💥 Impact

➡️ False takedowns  
➡️ Trust abuse

---

# 8️⃣ No Rate Limit on Port 22 (SSH)

### 🎯 Goal

Brute-force SSH.

### 🧪 Steps

1. Scan port 22
    
2. Try many passwords
    
3. No temporary ban
    

### 💥 Impact

➡️ **Server compromise**

---

# 🔓 Rate Limit Bypass via Headers

Some apps trust IP headers blindly.

### 🧪 Common Bypass Headers

```http
X-Forwarded-For: 127.0.0.1
X-Forwarded-Host: 127.0.0.1
X-Origination-IP: 0.0.0.0
X-Fowarded-For: 127.0.0.1
X-Remote-IP: 127.0.0.1
X-Remote-Addr: 127.0.0.1
```

### 🧪 Example Request

```http
POST /login.php HTTP/1.1
Host: target.com
X-Forwarded-For: 127.0.0.1

username=admin&password=FUZZ
```

---

# 🔁 Status Code Bypass (429 → 403)

### Meaning

- `429` = Too Many Requests
    
- App blocks but **logic is weak**
    

### 🧪 Test

- Add headers
    
- Change IP header
    
- Try again
    

If requests continue → **Bypass confirmed**

---

# 🛠 ffuf Example

```bash
ffuf -u https://example.com/login.php \
-w wordlist.txt \
-X POST \
-d "username=admin&password=FUZZ" \
-H "X-Forwarded-For: 127.0.0.1"
```

If responses continue without block → Vulnerable ✅

---
