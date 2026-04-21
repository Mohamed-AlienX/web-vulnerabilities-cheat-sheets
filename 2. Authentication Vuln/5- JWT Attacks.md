



# 🔐 JWT Attacks Cheat Sheet (Step-by-Step)

## 0️⃣ Recon – Identify JWT Usage (Always First)

**Goal:** Confirm the app uses JWT and how it’s verified.

### Steps

1. Login with valid credentials
    
2. Check request headers or cookies:
    
    ```
    Authorization: Bearer eyJ...
    OR
    Cookie: session=eyJ...
    ```
    
3. Decode JWT (Burp / jwt.io)
    
4. Note:
    
    - `alg`
        
    - `kid`
        
    - `jku`
        
    - `jwk`
        
    - claims like `sub`, `role`, `admin`
        

---

# 🟥 1️⃣  `alg: none` Signature Bypass

**Impact: CRITICAL**

### Idea

Server accepts **unsigned JWTs**

---

### Steps

1. Decode JWT
    
2. Change payload:
    
    ```json
    { "sub": "administrator" }
    ```
    
3. Change header:
    
    ```json
    { "alg": "none" }
    ```
    
4. Remove signature but keep trailing dot:
    
    ```
    header.payload.
    ```
    
5. Send request to `/admin`
    

---

### Impact

- No signature verification
    
- Anyone can forge tokens
    

---

# 🟥 3️⃣ Weak Secret Key (HS256 Bruteforce)

**Impact: HIGH → CRITICAL**

### Idea

JWT secret is weak (e.g. `secret`, `password`, `secret1`)

---

### Steps

1. Copy JWT
    
2. Bruteforce secret:
    
    ```bash
    hashcat -a 0 -m 16500 jwt.txt wordlist.txt
    ```
    
3. Re-sign JWT with cracked secret
    
4. Modify payload:
    
    ```json
    { "sub": "administrator" }
    ```
    
5. Access admin panel
    

---

### Impact

- Token forgery
    
- Privilege escalation
    
- Session takeover
    

---

# 🟥 4️⃣ `jwk` Header Injection

**Impact: HIGH**

### Idea

Server trusts **embedded public key** inside JWT.

---

### Steps

1. Generate your own RSA key
    
2. Modify payload → admin
    
3. Embed public key in header:
    
    ```json
    "jwk": { your public key }
    ```
    
4. Sign JWT using your private key
    
5. Send request
    

---

### Impact

- Attacker controls verification key
    
- Admin access without server key
    

---

# 🟥 5️⃣ `jku` Header Injection

**Impact: HIGH**

### Idea

Server fetches public key from attacker-controlled URL.

---

### Steps

1. Generate RSA key
    
2. Host public key as JWK Set on exploit server
    
3. Modify JWT header:
    
    ```json
    "jku": "https://attacker.com/jwks.json"
    "kid": "<matching kid>"
    ```
    
4. Change payload → administrator
    
5. Sign with your private key
    
6. Send request
    

---

### Impact

- External key trust abuse
    
- Full auth bypass
    

---

# 🟥 JWT Authentication Bypass via Algorithm Confusion

**Severity: CRITICAL**

### Vulnerability Summary

The application trusts the `alg` value from the JWT header, allowing an attacker to switch from `RS256` to `HS256` and sign a forged token using the RSA public key as an HMAC secret.

---

### Steps to Reproduce

1. Login as a normal user and capture the JWT.
    
2. Access `/jwks.json` to obtain the RSA public key.
    
3. Convert the public key to PEM and Base64 encode it.
    
4. Create a symmetric (HS256) key using the Base64 PEM as the secret.
    
5. Change JWT header:
    
    `"alg": "HS256"`
    
6. Modify payload:
    
    `"sub": "administrator"`
    
7. Sign the JWT using HS256 with the public key as secret.
    
8. Send request to `/admin`.
    

---

### Impact

Full authentication bypass and privilege escalation leading to admin account takeover.

---

### Root Cause

The server does not enforce a fixed JWT algorithm and reuses the RSA public key for HMAC verification.


---

# 🟧 6️⃣ `kid` Path Traversal

**Impact: MEDIUM → HIGH**

### Idea

Server loads key from filesystem using `kid`.

---

### Steps

1. Set header:
    
    ```json
    "kid": "../../../../../../dev/null"
    ```
    
1. Replace the generated value for the `k` property with a Base64-encoded null byte (`AA==`)
    
2. Modify payload → admin
    
3. Send request
    

---

### Impact

- Arbitrary file read
    
- Signature bypass
    

---

# 🟨 7️⃣ Missing Claim Validation

**Impact: MEDIUM**

### Examples

- No check on:
    
    - `exp`
        
    - `nbf`
        
    - `aud`
        
    - `iss`
        

---

### Steps

1. Remove or modify claims
    
2. Reuse expired tokens
    
3. Access protected endpoints
    

---

### Impact

- Session reuse
    
- Token replay
    
- Authorization flaws
    

---

# 📊 Impact Ranking (High → Low)

|Attack|Impact|
|---|---|
|Algorithm Confusion|🔥🔥🔥 Critical|
|alg:none|🔥🔥🔥 Critical|
|Weak Secret|🔥🔥 High|
|jku Injection|🔥🔥 High|
|jwk Injection|🔥🔥 High|
|kid Traversal|🔥 Medium|
|Claim Issues|🟡 Medium|

---

# 🛡️ Developer Fix Summary (For Reports)

- Enforce fixed algorithm (no token override)
    
- Separate RSA & HMAC verification logic
    
- Never trust `jku`, `jwk`, `kid` from user input
    
- Use strong secrets
    
- Validate all claims strictly
    

---
